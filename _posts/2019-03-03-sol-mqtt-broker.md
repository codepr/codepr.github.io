---
layout: post
title: "Sol - An MQTT broker from scratch. Part 1 - The protocol"
description: "Writing an MQTT broker from scratch, to really understand something you have to build it."
tags: [c, unix, tutorial]
---

It's been a while that for my daily work I have to deal with IoT architectures
and researching best patterns to develop such systems, including diving through
standards and protocols like MQTT; as I always been craving for new ideas to
learn and refine my programming skills, I thought that going a little deeper on
the topic would be cool and useful too. So once again I `git init` a low-level
project on my box to deepen my knowledge on the MQTT protocol, instead of the
usual Key-Value store, which happens to be my favourite pet-project of choice
when it comes to learn a new language or to dusting out on low-level system
programming on UNIX.

`Sol` will be a C project, a super-simple MQTT broker for Linux platform which
will support version 3.1.1 of the protocol, skipping on older protocols for
now, very similar to a lightweight mosquitto (which is already a lightweight
piece of software anyway). As a side note, the name decision is a 50/50 for the
elegance i feel for short names and the martian day (The Martian docet). Or
maybe it stands for Shitty Obnoxious Laxative. Tastes.


Going per steps, I usually init my C projects in order to have all sources
in a single folder:

{% highlight bash %}
sol/
 ├── src/
 ├── CHANGELOG
 ├── CMakeLists.txt
 ├── COPYING
 └── README.md
{% endhighlight %}

Here the repository on Github
[https://github.com/codepr/sol.git](https://github.com/codepr/sol.git).  I'll
try to describe step by step my journey into the development of the software,
without being too much verbose, and listing lot of code directly with brief
explanation of its purpose. The best way still remains to write it down,
compile it and play/modify it.<br>
I'd like to underline that the resulting software will be a fully functioning
broker, but with large space for improvements and optimization as well as code
quality improvements and probably, with some hidden features as well (aka bugs
:P).

### General architecture

In essence a broker is a middleware, a software that accepts input from
multiple clients (producers) and forward it to a set of destinatary clients
(consumers) using an abstraction to define and manage these groups of clients
in the form of a channel or `topic` as it's called by the protocol standards.
Much like an IRC channel or equivalent in a generic chart, each consumer client
can subscribe to `topics` in order to receive all messages published by other
clients to those `topics`.

The first idea coming to mind is a server built on top of a data structure
of some kind that allow to easily manage these `topics` and connected
`clients`, being them producers or consumers. Each message received by a client
must be forwarded to all other connected clients that are subscribed to the
specified topic of the message.

Let's try this way, a TCP server and a module to handle binary communication
through the wire. There are many ways to implement a server, threads, fork
processes and multiplexing I/O, which is the way I'd like to explore the most.
We'll start with a single-threaded multiplexing I/O server, with the possibility
on future to scale it out using threads, **epoll** interface for multiplexing in
fact is thread-safe by implementation.

### The MQTT protocol

First of all, we have to model some structures to handle MQTT packets, by
following specifications on the [official
documentations](http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/errata01/os/mqtt-v3.1.1-errata01-os-complete.html).
Starting from the opcode table and the MQTT header, according to the docs every packet
consists of 3 parts:
- Fixed Header (mandatory)
- Variable Header (optional)
- Payload (optional)

The Fixed Header part consists of the first byte for command type and flags,
and a second to fifth byte to store the remaining length of the packet.

{% highlight markdown %}
# Fixed Header

 | Bit    | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
 |--------|---------------|---------------|
 | Byte 1 | MQTT type     |  Flags        |
 |--------|-------------------------------|
 | Byte 2 |                               |
 |  .     |      Remaning Length          |
 |  .     |                               |
 | Byte 5 |                               |

{% endhighlight %}

Flags are not all mandatory, just the 4 bits block `MQTT Control Type`, the
others are:

- `Dup` flag, used when a message is sent more than one time
- `QoS` level, can be `AT_MOST_ONCE`, `AT_LEAST_ONCE` and `EXACTLY_ONCE`, 0,
   1, 2 respectively
- `Retain` flag, if a message should be retained, in other words, when a
   message is published on a topic, it is saved and future connecting clients
   will receive it. It can be updated with another retained message.


So fire up Vim (or your favourite editor) and start writing `mqtt.h` header
file containing the Control Packet Types and a struct to handle the Fixed
Header:

**src/mqtt.h**
{% highlight c %}
#ifndef MQTT_H
#define MQTT_H


#define MQTT_HEADER_LEN 2
#define MQTT_ACK_LEN    4


/* Message types */
enum message_opcode {
    CONNECT     = 0x10,
    CONNACK     = 0x20,
    PUBLISH     = 0x30,
    PUBACK      = 0x40,
    PUBREC      = 0x50,
    PUBREL      = 0x60,
    PUBCOMP     = 0x70,
    SUBSCRIBE   = 0x80,
    SUBACK      = 0x90,
    UNSUBSCRIBE = 0xA0,
    UNSUBACK    = 0xB0,
    PINGREQ     = 0xC0,
    PINGRESP    = 0xD0,
    DISCONNECT  = 0xE0
};


enum message_type {
    CONNECT_TYPE     = 1,
    CONNACK_TYPE     = 2,
    PUBLISH_TYPE     = 3,
    PUBACK_TYPE      = 4,
    PUBREC_TYPE      = 5,
    PUBREL_TYPE      = 6,
    PUBCOMP_TYPE     = 7,
    SUBSCRIBE_TYPE   = 8,
    SUBACK_TYPE      = 9,
    UNSUBSCRIBE_TYPE = 10,
    UNSUBACK_TYPE    = 11,
    PINGREQ_TYPE     = 12,
    PINGRESP_TYPE    = 13,
    DISCONNECT_TYPE  = 14
};


enum qos_level { AT_MOST_ONCE, AT_LEAST_ONCE, EXACTLY_ONCE };


union mqtt_header {

    unsigned char byte;

    struct {
        unsigned retain : 1;
        unsigned qos : 2;
        unsigned dup : 1;
        unsigned type : 4;
    } bits;

};

{% endhighlight %}

The first 2 `#define` refers to fixed sizes of the MQTT Fixed Header and of
every type of MQTT `ACK` packets, set for convenience, we'll use those later.

As shown, we leverage the **union**, a value that may have any of several
representations withing the same position in memory, to represent a byte. In
other words, inside unions, in contrast to normal **struct**, there can be only
one field with a value. Their position in memory are shared, this way using
**bitfields** we can effectively manipulate single bits or portions of a byte.

The first Control Packet we're going to define is the `CONNECT`. The
`CONNECT` is the first packet that must be sent when a client establish a new
connection and it must be extactly one, more than one `CONNECT` per client must
be treated as a violation of the protocol and the client must be dropped.<br>
At each `CONNECT` must be followed in response a `CONNACK`.

**src/mqtt.h**

{% highlight c %}

struct mqtt_connect {

    union mqtt_header header;

    union {

        unsigned char byte;

        struct {
            int reserverd : 1;
            unsigned clean_session : 1;
            unsigned will : 1;
            unsigned will_qos : 2;
            unsigned will_retain : 1;
            unsigned password : 1;
            unsigned username : 1;
        } bits;
    };

    struct {
        unsigned short keepalive;
        unsigned char *client_id;
        unsigned char *username;
        unsigned char *password;
        unsigned char *will_topic;
        unsigned char *will_message;
    } payload;

};


struct mqtt_connack {

    union mqtt_header header;

    union {

        unsigned char byte;

        struct {
            unsigned session_present : 1;
            unsigned reserverd : 7;
        } bits;
    };

    unsigned char rc;
};

{% endhighlight %}

From now on, the definition of other packets are trivial by reproducing the
pattern, accordingly to the documentation of MQTT v3.1.1.

We proceed with `SUBSCRIBE`, `UNSUBSCRIBE` and `PUBLISH`. `SUBSCRIBE` is the
only packet with a dedicated packet definition `SUBACK`, the other can be
defined as generic `ACK`, and typenamed usng **typedef** for semantic
separation.

**src/mqtt.h**

{% highlight c %}

struct mqtt_subscribe {

    union mqtt_header header;

    unsigned short pkt_id;

    unsigned short tuples_len;

    struct {
        unsigned short topic_len;
        unsigned char *topic;
        unsigned qos;
    } *tuples;
};


struct mqtt_unsubscribe {

    union mqtt_header header;

    unsigned short pkt_id;

    unsigned short tuples_len;

    struct {
        unsigned short topic_len;
        unsigned char *topic;
    } *tuples;
};


struct mqtt_suback {

    union mqtt_header header;

    unsigned short pkt_id;

    unsigned short rcslen;

    unsigned char *rcs;
};


struct mqtt_publish {

    union mqtt_header header;

    unsigned short pkt_id;

    unsigned short topiclen;
    unsigned char *topic;
    unsigned short payloadlen;
    unsigned char *payload;
};


struct mqtt_ack {

    union mqtt_header header;

    unsigned short pkt_id;
};

{% endhighlight %}

The remaning `ACK` packets, namely:
- `PUBACK`
- `PUBREC`
- `PUBREL`
- `PUBCOMP`
- `UNSUBACK`
- `PINGREQ`
- `PINGRESP`
- `DISCONNECT`

can be obtained by typedef'ing `struct ack`, just for semantic separation of
concerns. The last one, `DISCONNECT`, is not really an `ACK` but the format is
the same.

**src/mqtt.h**

{% highlight c %}

typedef struct mqtt_ack mqtt_puback;
typedef struct mqtt_ack mqtt_pubrec;
typedef struct mqtt_ack mqtt_pubrel;
typedef struct mqtt_ack mqtt_pubcomp;
typedef struct mqtt_ack mqtt_unsuback;
typedef union mqtt_header mqtt_pingreq;
typedef union mqtt_header mqtt_pingresp;
typedef union mqtt_header mqtt_disconnect;

{% endhighlight %}

We can finally define a generic MQTT packet as a `union` of the previously
defined packets.

**src/mqtt.h**

{% highlight c %}

union mqtt_packet {

    struct mqtt_ack ack;
    union mqtt_header header;
    struct mqtt_connect connect;
    struct mqtt_connack connack;
    struct mqtt_suback suback;
    struct mqtt_publish publish;
    struct mqtt_subscribe subscribe;
    struct mqtt_unsubscribe unsubscribe;

};

{% endhighlight %}

We proceed now with the definition of some public functions, here in the header
we want to collect only those functions and structure that should be used by
other modules.

To handle the communication using the MQTT protocol we need essentially 4 functions,
2 for each direction of the interaction between server and client:

- A packing function (serializing or marshalling, i won't dive here in a
  dissertion on the correct usage of the terms)
- An unpacking function (deserializing/unmarshalling)

Supported by 2 functions to handle the encoding and decoding of the
Remaning Length in the Fixed Header part.

**src/mqtt.h**

{% highlight c %}

int mqtt_encode_length(unsigned char *, size_t);

unsigned long long mqtt_decode_length(const unsigned char **);

int unpack_mqtt_packet(const unsigned char *, union mqtt_packet *);

unsigned char *pack_mqtt_packet(const union mqtt_packet *, unsigned);

{% endhighlight %}

We also add some utility functions to build packets and to release
heap-alloc'ed ones, nothing special here.

**src/mqtt.h**

{% highlight c %}

union mqtt_header *mqtt_packet_header(unsigned char);

struct mqtt_ack *mqtt_packet_ack(unsigned char , unsigned short);

struct mqtt_connack *mqtt_packet_connack(unsigned char ,
                                         unsigned char ,
                                         unsigned char);

struct mqtt_suback *mqtt_packet_suback(unsigned char, unsigned short,
                                       unsigned char *, unsigned short);

struct mqtt_publish *mqtt_packet_publish(unsigned char, unsigned short, size_t,
                                         unsigned char *,
                                         size_t, unsigned char *);

void mqtt_packet_release(union mqtt_packet *, unsigned);


#endif
{% endhighlight %}

Fine. We have a decent header module that define all that we need for handling
the communication using the protocol. Let's now implement those functions.
First of all we define some "private" helpers, to pack and unpack each MQTT
packet, these will be called by the previously defined "public" functions
`unpack_mqtt_packet` and `pack_mqtt_packet`.

**src/mqtt.c**

{% highlight c %}

#include "mqtt.h"


static size_t unpack_mqtt_connect(const unsigned char *,
                                  union mqtt_header *,
                                  union mqtt_packet *);

static size_t unpack_mqtt_publish(const unsigned char *,
                                  union mqtt_header *,
                                  union mqtt_packet *);

static size_t unpack_mqtt_subscribe(const unsigned char *,
                                    union mqtt_header *,
                                    union mqtt_packet *);

static size_t unpack_mqtt_unsubscribe(const unsigned char *,
                                      union mqtt_header *,
                                      union mqtt_packet *);

static size_t unpack_mqtt_ack(const unsigned char *,
                              union mqtt_header *,
                              union mqtt_packet *);


static unsigned char *pack_mqtt_header(const union mqtt_header *);

static unsigned char *pack_mqtt_ack(const union mqtt_packet *);

static unsigned char *pack_mqtt_connack(const union mqtt_packet *);

static unsigned char *pack_mqtt_suback(const union mqtt_packet *);

static unsigned char *pack_mqtt_publish(const union mqtt_packet *);

{% endhighlight %}

### Packing and unpacking

Before continuing with the implementation of all defined functions on
`src/mqtt.h`, we need to implement some helpers functions to ease the pack and
unpack process of each received packet and also ready for send forged MQTT
packet.

Let's go fast here, it's just simple serialization/deserialization respecting
the network byte order of the packets.

**src/pack.h**

{% highlight c %}

#ifndef PACK_H
#define PACK_H

#include <stdio.h>
#include <stdint.h>

/* To network byte order for long long int */
void htonll(uint8_t *, uint_least64_t );

/* To host from network byte order for long long int */
uint_least64_t ntohll(const uint8_t *);

/* Reading data on const uint8_t pointer */
// bytes -> uint8_t
uint8_t unpack_u8(const uint8_t **);

// bytes -> uint16_t
uint16_t unpack_u16(const uint8_t **);

// bytes -> uint32_t
uint32_t unpack_u32(const uint8_t **);

// bytes -> uint64_t
uint64_t unpack_u64(const uint8_t **);

// read a defined len of bytes
uint8_t *unpack_bytes(const uint8_t **, size_t, uint8_t *);

/* Write data on const uint8_t pointer */
// append a uint8_t -> bytes into the bytestring
void pack_u8(uint8_t **, uint8_t);

// append a uint16_t -> bytes into the bytestring
void pack_u16(uint8_t **, uint16_t);

// append a uint32_t -> bytes into the bytestring
void pack_u32(uint8_t **, uint32_t);

// append a uint64_t -> bytes into the bytestring
void pack_u64(uint8_t **, uint64_t);

// append len bytes into the bytestring
void pack_bytes(uint8_t **, uint8_t *);


#endif

{% endhighlight %}

And the corresponding implementation

**src/pack.c**

{% highlight c %}

#include <string.h>
#include <arpa/inet.h>
#include "pack.h"


/* Host-to-network (native endian to big endian) */
void htonll(uint8_t *block, uint_least64_t num) {
    block[0] = num >> 56 & 0xFF;
    block[1] = num >> 48 & 0xFF;
    block[2] = num >> 40 & 0xFF;
    block[3] = num >> 32 & 0xFF;
    block[4] = num >> 24 & 0xFF;
    block[5] = num >> 16 & 0xFF;
    block[6] = num >> 8 & 0xFF;
    block[7] = num >> 0 & 0xFF;
}

/* Network-to-host (big endian to native endian) */
uint_least64_t ntohll(const uint8_t *block) {
    return (uint_least64_t) block[0] << 56 | (uint_least64_t) block[1] << 48
        | (uint_least64_t) block[2] << 40 | (uint_least64_t) block[3] << 32
        | (uint_least64_t) block[4] << 24 | (uint_least64_t) block[5] << 16
        | (uint_least64_t) block[6] << 8 | (uint_least64_t) block[7] << 0;
}

// Reading data
uint8_t unpack_u8(const uint8_t **buf) {
    uint8_t val = **buf;
    (*buf)++;
    return val;
}


uint16_t unpack_u16(const uint8_t **buf) {
    uint16_t val = ntohs(*((uint16_t *) (*buf)));
    (*buf) += sizeof(uint16_t);
    return val;
}


uint32_t unpack_u32(const uint8_t **buf) {
    uint32_t val = ntohl(*((uint32_t *) (*buf)));
    (*buf) += sizeof(uint32_t);
    return val;
}


uint64_t unpack_u64(const uint8_t **buf) {
    uint64_t val = ntohll(*buf);
    (*buf) += sizeof(uint64_t);
    return val;
}


uint8_t *unpack_bytes(const uint8_t **buf, size_t len, uint8_t *str) {

    memcpy(str, *buf, len);
    str[len] = '\0';
    (*buf) += len;

    return str;
}

// Write data
void pack_u8(uint8_t **buf, uint8_t val) {
    **buf = val;
    (*buf) += sizeof(uint8_t);
}


void pack_u16(uint8_t **buf, uint16_t val) {
    *((uint16_t *) (*buf)) = htons(val);
    (*buf) += sizeof(uint16_t);
}


void pack_u32(uint8_t **buf, uint32_t val) {
    *((uint32_t *) (*buf)) = htonl(val);
    (*buf) += sizeof(uint32_t);
}


void pack_u64(uint8_t **buf, uint64_t val) {
    htonll(*buf, val);
    (*buf) += sizeof(uint64_t);
}


void pack_bytes(uint8_t **buf, uint8_t *str) {

    size_t len = strlen((char *) str);

    memcpy(*buf, str, len);
    (*buf) += len;
}

{% endhighlight %}

This allow us to handle incoming stream of bytes and forge them to respond to
connected clients. Let's move one.

### Back to src/mqtt.c

The first step will be the implemetation of the Fixed Header Remaning Length
functions. The MQTT documentation suggests a pseudo-code implementation in one
of the first paragraphs, we'll stick to that, it's fairly simple and clear.
We'll see why and how after the first byte of the Fixed Header, the next 1 or
2 or 3 or 4 bytes are used to encode the remainig bytes of the packet.

> The Remaining Length is the number of bytes remaining within the current
> packet, including data in the variable header and the payload. The Remaining
> Length does not include the bytes used to encode the Remaining Length.
>
> The Remaining Length is encoded using a variable length encoding scheme which
> uses a single byte for values up to 127. Larger values are handled as
> follows. The least significant seven bits of each byte encode the data, and
> the most significant bit is used to indicate that there are following bytes
> in the representation. Thus each byte encodes 128 values and a "continuation
> bit". The maximum number of bytes in the Remaining Length field is four.

No need for further explanation, the MQTT documentation is crystal clear.

**src/mqtt.c**

{% highlight c %}

/*
 * Encode Remaining Length on a MQTT packet header, comprised of Variable
 * Header and Payload if present. It does not take into account the bytes
 * required to store itself. Refer to MQTT v3.1.1 algorithm for the
 * implementation.
 */
int mqtt_encode_length(unsigned char *buf, size_t len) {

    int bytes = 0;

    do {

        if (bytes + 1 > MAX_LEN_BYTES)
            return bytes;

        char d = len % 128;
        len /= 128;

        /* if there are more digits to encode, set the top bit of this digit */
        if (len > 0)
            d |= 0x80;

        buf[bytes++] = d;

    } while (len > 0);

    return bytes;
}

/*
 * Decode Remaining Length comprised of Variable Header and Payload if
 * present. It does not take into account the bytes for storing length. Refer
 * to MQTT v3.1.1 algorithm for the implementation suggestion.
 *
 * TODO Handle case where multiplier > 128 * 128 * 128
 */
unsigned long long mqtt_decode_length(const unsigned char **buf) {

    char c;
    int multiplier = 1;
    unsigned long long value = 0LL;

    do {
        c = **buf;
        value += (c & 127) * multiplier;
        multiplier *= 128;
        (*buf)++;
    } while ((c & 128) != 0);

    return value;
}

{% endhighlight %}

Now we can read the first header byte and the total length of the packet. Let's
move on with the unpacking of the `CONNECT` packet.
It's the packet with more flags and the second one in length behind only the
`PUBLISH` packet.

It consists in:
- A fixed header with MQTT Control packet type 1 on the first 4 most
  significant bits (MSB from now on) and unused (reserved for future
  implementation) for the 4 least significant bits (LSB).
- The Remaining Length of the packet
- The Variable Header which consists of four fields:
    - Protocol Name
    - Protocol Level
    - Connect Flags
    - Keep Alive

> The Protocol Name is a UTF-8 encoded string that represents the protocol name
> “MQTT”, capitalized. The string, its offset and length will not be changed by
> future versions of the MQTT specification.

For version 3.1.1 the Protocol Name is 'M' 'Q' 'T' 'T', 4 bytes in total, we
will ignore for now what is the name for older versions.<br>
Connect flags byte contains some indications on the behaviour of the client and
the presence or absence of fields in the payload:

- Username flag
- Password flag
- Will retain
- Will QoS
- Will flag
- Clean Session

The last bit is reserved for future implementations. All other flags are
intended as booleans, on the payload part of the packet, according to those
flags there are also the corresponding fields, so let's say we have username
and password at true, on the payload we'll find a 2 bytes field representing
username length, followed by the username itself, and the same for the password.

**src/mqtt.c**

{% highlight c %}

/*
 * MQTT unpacking functions
 */

static size_t unpack_mqtt_connect(const unsigned char *raw,
                                  union mqtt_header *hdr,
                                  union mqtt_packet *pkt) {

    struct mqtt_connect connect = { .header = *hdr };
    pkt->connect = connect;

    const unsigned char *init = raw;
    /*
     * Second byte of the fixed header, contains the length of remaining bytes
     * of the connect packet
     */
    size_t len = mqtt_decode_length(&raw);

    /*
     * For now we ignore checks on protocol name and reserverd bits, just skip
     * to the 8th byte
     */
    raw = init + 8;

    /* Read variable header byte flags */
    pkt->connect.byte = unpack_u8((const uint8_t **) &raw);

    /* Read keepalive MSB and LSB (2 bytes word) */
    pkt->connect.payload.keepalive = unpack_u16((const uint8_t **) &raw);

    /* Read CID length (2 bytes word) */
    uint16_t cid_len = unpack_u16((const uint8_t **) &raw);

    /* Read the client id */
    if (cid_len > 0) {
        pkt->connect.payload.client_id = malloc(cid_len + 1);
        unpack_bytes((const uint8_t **) &raw, cid_len,
                     pkt->connect.payload.client_id);
    }

    /* Read the will topic and message if will is set on flags */
    if (pkt->connect.bits.will == 1) {

        uint16_t will_topic_len = unpack_u16((const uint8_t **) &raw);
        pkt->connect.payload.will_topic = malloc(will_topic_len + 1);
        unpack_bytes((const uint8_t **) &raw, will_topic_len,
                     pkt->connect.payload.will_topic);

        uint16_t will_message_len = unpack_u16((const uint8_t **) &raw);
        pkt->connect.payload.will_message = malloc(will_message_len + 1);
        unpack_bytes((const uint8_t **) &raw, will_message_len,
                     pkt->connect.payload.will_message);
    }

    /* Read the username if username flag is set */
    if (pkt->connect.bits.username == 1) {
        uint16_t username_len = unpack_u16((const uint8_t **) &raw);
        pkt->connect.payload.username = malloc(username_len + 1);
        unpack_bytes((const uint8_t **) &raw, username_len,
                     pkt->connect.payload.username);
    }

    /* Read the password if password flag is set */
    if (pkt->connect.bits.password == 1) {
        uint16_t password_len = unpack_u16((const uint8_t **) &raw);
        pkt->connect.payload.password = malloc(password_len + 1);
        unpack_bytes((const uint8_t **) &raw, password_len,
                     pkt->connect.payload.password);
    }

    return len;
}

{% endhighlight %}

The `PUBLISH` packet now:

{% highlight markdown %}

 |   Bit    |  7  |  6  |  5  |  4  |  3  |  2  |  1  |   0    |  <-- Fixed Header
 |----------|-----------------------|--------------------------|
 | Byte 1   | MQTT type 3           | dup |    QoS    | retain |
 |----------|--------------------------------------------------|
 | Byte 2   |                                                  |
 |  .       |               Remaning Length                    |
 |  .       |                                                  |
 | Byte 5   |                                                  |
 |----------|--------------------------------------------------|  <-- Variable Header
 | Byte 6   |                Topic len MSB                     |
 | Byte 7   |                Topic len LSB                     |
 |-------------------------------------------------------------|
 | Byte 8   |                Topic name                        |
 | Byte N   |                                                  |
 |----------|--------------------------------------------------|
 | Byte N+1 |            Packet Identifier MSB                 |
 | Byte N+2 |            Packet Identifier LSB                 |
 |----------|--------------------------------------------------|  <-- Payload
 | Byte N+3 |                   Payload                        |
 | Byte N+M |                                                  |

{% endhighlight %}


Packet identifier MSB and LSB are present in the packet if and only if the QoS
level is > 0, with a QoS set to *at most once* there's no need for a packet ID.
Payload length is calculated by subtracting the Remaining Length with all the
other fields already unpacked.

**src/mqtt.c**

{% highlight c %}

static size_t unpack_mqtt_publish(const unsigned char *raw,
                                  union mqtt_header *hdr,
                                  union mqtt_packet *pkt) {

    struct mqtt_publish publish = { .header = *hdr };
    pkt->publish = publish;

    /*
     * Second byte of the fixed header, contains the length of remaining bytes
     * of the connect packet
     */
    size_t len = mqtt_decode_length(&raw);

    /* Read topic length and topic of the soon-to-be-published message */
    uint16_t topic_len = unpack_u16((const uint8_t **) &raw);
    pkt->publish.topiclen = topic_len;
    pkt->publish.topic = malloc(topic_len + 1);
    unpack_bytes((const uint8_t **) &raw, topic_len, pkt->publish.topic);

    uint16_t message_len = len;

    /* Read packet id */
    if (publish.header.bits.qos > AT_MOST_ONCE) {
        pkt->publish.pkt_id = unpack_u16((const uint8_t **) &raw);
        message_len -= sizeof(uint16_t);
    }

    /*
     * Message len is calculated subtracting the length of the variable header
     * from the Remaining Length field that is in the Fixed Header
     */
    message_len -= (sizeof(uint16_t) + topic_len);
    pkt->publish.payloadlen = message_len;
    pkt->publish.payload = malloc(message_len + 1);
    unpack_bytes((const uint8_t **) &raw, message_len, pkt->publish.payload);

    return len;
}

{% endhighlight %}

Subscribe and unsubscribe packets are fairly similar, they reflect the
`PUBLISH` packet, but for payload they have a list of tuple consisting in a
pair (topic, QoS). Their implementation is practically identical, with the
only difference in the payload part, where `UNSUBSCRIBE` doesn't specify QoS
for each topic.

{% highlight c %}

static size_t unpack_mqtt_subscribe(const unsigned char *raw,
                                    union mqtt_header *hdr,
                                    union mqtt_packet *pkt) {

    struct mqtt_subscribe subscribe = { .header = *hdr };

    /*
     * Second byte of the fixed header, contains the length of remaining bytes
     * of the connect packet
     */
    size_t len = mqtt_decode_length(&raw);
    size_t remaining_bytes = len;

    /* Read packet id */
    subscribe.pkt_id = unpack_u16((const uint8_t **) &raw);
    remaining_bytes -= sizeof(uint16_t);

    /*
     * Read in a loop all remaning bytes specified by len of the Fixed Header.
     * From now on the payload consists of 3-tuples formed by:
     *  - topic length
     *  - topic filter (string)
     *  - qos
     */
    int i = 0;
    while (remaining_bytes > 0) {

        /* Read length bytes of the first topic filter */
        uint16_t topic_len = unpack_u16((const uint8_t **) &raw);
        remaining_bytes -= sizeof(uint16_t);

        /* We have to make room for additional incoming tuples */
        subscribe.tuples = realloc(subscribe.tuples,
                                   (i+1) * sizeof(*subscribe.tuples));
        subscribe.tuples[i].topic_len = topic_len;
        subscribe.tuples[i].topic = malloc(topic_len + 1);
        unpack_bytes((const uint8_t **) &raw, topic_len,
                     subscribe.tuples[i].topic);
        remaining_bytes -= topic_len;
        subscribe.tuples[i].qos = unpack_u8((const uint8_t **) &raw);
        remaining_bytes -= sizeof(uint8_t);
        i++;
    }

    subscribe.tuples_len = i;

    pkt->subscribe = subscribe;

    return len;
}


static size_t unpack_mqtt_unsubscribe(const unsigned char *raw,
                                      union mqtt_header *hdr,
                                      union mqtt_packet *pkt) {

    struct mqtt_unsubscribe unsubscribe = { .header = *hdr };

    /*
     * Second byte of the fixed header, contains the length of remaining bytes
     * of the connect packet
     */
    size_t len = mqtt_decode_length(&raw);
    size_t remaining_bytes = len;

    /* Read packet id */
    unsubscribe.pkt_id = unpack_u16((const uint8_t **) &raw);
    remaining_bytes -= sizeof(uint16_t);

    /*
     * Read in a loop all remaning bytes specified by len of the Fixed Header.
     * From now on the payload consists of 2-tuples formed by:
     *  - topic length
     *  - topic filter (string)
     */
    int i = 0;
    while (remaining_bytes > 0) {

        /* Read length bytes of the first topic filter */
        uint16_t topic_len = unpack_u16((const uint8_t **) &raw);
        remaining_bytes -= sizeof(uint16_t);

        /* We have to make room for additional incoming tuples */
        unsubscribe.tuples = realloc(unsubscribe.tuples,
                                     (i+1) * sizeof(*unsubscribe.tuples));
        unsubscribe.tuples[i].topic_len = topic_len;
        unsubscribe.tuples[i].topic = malloc(topic_len + 1);
        unpack_bytes((const uint8_t **) &raw, topic_len,
                     unsubscribe.tuples[i].topic);
        remaining_bytes -= topic_len;

        i++;
    }

    unsubscribe.tuples_len = i;

    pkt->unsubscribe = unsubscribe;

    return len;
}

{% endhighlight %}

And finally the `ACK`. In MQTT doesn't exists generic acks, but the structure
of most of the ack-like packets is practically the same for each one, formed by
a header and a packet ID.

These are:

- `PUBACK`
- `PUBREC`
- `PUBREL`
- `PUBCOMP`
- `UNSUBACK`

{% highlight c %}

static size_t unpack_mqtt_ack(const unsigned char *raw,
                              union mqtt_header *hdr,
                              union mqtt_packet *pkt) {

    struct mqtt_ack ack = { .header = *hdr };

    /*
     * Second byte of the fixed header, contains the length of remaining bytes
     * of the connect packet
     */
    size_t len = mqtt_decode_length(&raw);

    ack.pkt_id = unpack_u16((const uint8_t **) &raw);
    pkt->ack = ack;

    return len;
}

{% endhighlight %}

We have now all needed helpers functions to implement our only exposed function
on the header `mqtt_mqtt_packet`. It ended up being a fairly short and simple
function, we'll use a static array to map all helper functions, making it O(1)
the selection of the correct unpack function based on the Control Packet type.

{% highlight c %}

/*
 * Unpack functions mapping unpacking_handlers positioned in the array based
 * on message type
 */
static mqtt_unpack_handler *unpack_handlers[11] = {
    NULL,
    unpack_mqtt_connect,
    NULL,
    unpack_mqtt_publish,
    unpack_mqtt_ack,
    unpack_mqtt_ack,
    unpack_mqtt_ack,
    unpack_mqtt_ack,
    unpack_mqtt_subscribe,
    NULL,
    unpack_mqtt_unsubscribe
};


int unpack_mqtt_packet(const unsigned char *raw, union mqtt_packet *pkt) {

    int rc = 0;

    /* Read first byte of the fixed header */
    unsigned char type = *raw;

    union mqtt_header header = {
        .byte = type
    };

    if (header.bits.type == DISCONNECT_TYPE
        || header.bits.type == PINGREQ_TYPE
        || header.bits.type == PINGRESP_TYPE)
        pkt->header = header;
    else
        /* Call the appropriate unpack handler based on the message type */
        rc = unpack_handlers[header.bits.type](++raw, &header, pkt);

    return rc;
}

{% endhighlight %}

The first part ends here, at this point we have two modules, one of utility for
general serialization operations and one to handle the protocol itself accordingly
to the standard defined by OASIS.

{% highlight bash %}
sol/
 ├── src/
 │    ├── mqtt.h
 |    ├── mqtt.c
 │    ├── pack.h
 │    └── pack.c
 ├── CHANGELOG
 ├── CMakeLists.txt
 ├── COPYING
 └── README.md
{% endhighlight %}

Just `git commit` and `git push`. Cya.

[Sol - An MQTT broker from scratch. Part-2](sol-mqtt-broker-p2) will deal with the
networking utilities needed to setup our communication layer and thus the server.
