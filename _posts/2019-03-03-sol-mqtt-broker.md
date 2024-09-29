---
layout: post
title: "Sol - An MQTT broker from scratch. Part 1 - The protocol"
description: "Writing an MQTT broker from scratch, to really understand something you have to build it."
categories: c unix tutorial
---

It's been a while that for my daily work I deal with IoT architectures and
research best patterns to develop such systems, including diving through
standards and protocols like MQTT;
<!--more-->
as I always been craving for new ideas to learn and refine my programming
skills, I thought that going a little deeper on the topic would've been cool
and useful too. So once again I `git init` a low-level project on my box
pushing myself a little further by sharing my steps.

**Sol** will be a C project, a super-simple MQTT broker targeting Linux
platform which will support version 3.1.1 of the protocol, skipping older
versions for now, very similar to a lightweight Mosquitto (which is already a
lightweight piece of software anyway), and with the abundance of MQTT clients
out there, testing will be fairly easy. The final result will hopefully serve
as a base for something more clean and with more features, what we're going to
create have to be considered a minimal implementation, an MVP. As a side note,
the name decision is a 50/50 for the elegance i feel for short names and the
martian day (The Martian docet). Or maybe it stands for **S**crappy **O**l'
**L**oser. Tastes.

**Note**: the project won't compile till the very end of the series, following
all steps, to test single parts and modules I suggest to provide a main by
yourself and stop to make experiments, change parts etc.


Step by step, I usually init my C projects in order to have all sources
in a single folder:

```bash
sol/
 ├── src/
 ├── CHANGELOG
 ├── CMakeLists.txt
 ├── COPYING
 └── README.md
```

Here the [repository](https://github.com/codepr/sol/tree/tutorial) on GitHub.
I'll try to describe step by step my journey into the development of the software,
without being too verbose, and listing lot of code directly with brief
explanations of its purpose. The best way is still to write it down,
compile and/or play/modify it.<br>
This will be a series of posts, each one tackling and mostly implementing a single
concept/module of the project:

- [Part 1 - Protocol](../sol-mqtt-broker), lays the foundations to handle the MQTT protocol packets
- [Part 2 - Networking](../sol-mqtt-broker-p2), utility module, focus on network communication
- [Part 3 - Server](../sol-mqtt-broker-p3), the main entry point of the program
- [Part 4 - Data structures](../sol-mqtt-broker-p4), utility and educational modules
- [Part 5 - Topic abstraction](../sol-mqtt-broker-p5), handling the main scaling and grouping abstraction of the broker
- [Part 6 - Handlers](../sol-mqtt-broker-p6), server completion, for each packet there's a handler dedicated
- [Bonus - Multi-threading](../sol-mqtt-broker-bonus), improvements, bug fixing and integration of multi-threading model

I'd like to underline that the resulting software will be a fully functioning
broker, but with large space for improvements and optimization as well as code
quality improvements and probably, with some hidden features as well (aka bugs
:P).

### General architecture

In essence a broker is a middleware, a software that accepts input from
multiple clients (producers) and forward it to a set of recipient clients
(consumers) using an abstraction to define and manage these groups of clients
in the form of a channel, or **topic**, as it's called by the protocol standards.
Much like an IRC channel or equivalent in a generic chat, each consumer client
can subscribe to **topics** in order to receive all messages published by other
clients to those **topics**.

The first idea coming to mind is a server built on top of a data structure
of some kind that allow to easily manage these **topics** and connected
clients, being them producers or consumers. Each message received by a client
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
documentation](http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/errata01/os/mqtt-v3.1.1-errata01-os-complete.html).
Starting from the opcode table and the MQTT header, according to the docs,
every packet consists of 3 parts:
- Fixed Header (mandatory)
- Variable Header (optional)
- Payload (optional)

The Fixed Header part consists of the first byte for command type and flags,
and a second to fifth byte to store the remaining length of the packet.

```markdown
# Fixed Header

 | Bit    | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
 |--------|---------------|---------------|
 | Byte 1 | MQTT type     |  Flags        |
 |--------|-------------------------------|
 | Byte 2 |                               |
 |  .     |      Remaining Length         |
 |  .     |                               |
 | Byte 5 |                               |

```

Flags are not all mandatory, just the 4 bits block MQTT Control Type, the
others are:

- Dup flag, used when a message is sent more than one time
- QoS level, can be AT_MOST_ONCE, AT_LEAST_ONCE and EXACTLY_ONCE, 0,
  1, 2 respectively
- Retain flag, if a message should be retained, in other words, when a
  message is published on a topic, it is saved and future connecting clients
  will receive it. It can be updated with another retained message.


So fire up Vim (or your favourite editor) and start writing `mqtt.h` header
file containing the Control Packet Types and a struct to handle the Fixed
Header:

<hr>
**src/mqtt.h**
<hr>
```c
#ifndef MQTT_H
#define MQTT_H

#include <stdio.h>

#define MQTT_HEADER_LEN 2
#define MQTT_ACK_LEN    4

/*
 * Stub bytes, useful for generic replies, these represent the first byte in
 * the fixed header
 */
#define CONNACK_BYTE  0x20
#define PUBLISH_BYTE  0x30
#define PUBACK_BYTE   0x40
#define PUBREC_BYTE   0x50
#define PUBREL_BYTE   0x60
#define PUBCOMP_BYTE  0x70
#define SUBACK_BYTE   0x90
#define UNSUBACK_BYTE 0xB0
#define PINGRESP_BYTE 0xD0

/* Message types */
enum packet_type {
    CONNECT     = 1,
    CONNACK     = 2,
    PUBLISH     = 3,
    PUBACK      = 4,
    PUBREC      = 5,
    PUBREL      = 6,
    PUBCOMP     = 7,
    SUBSCRIBE   = 8,
    SUBACK      = 9,
    UNSUBSCRIBE = 10,
    UNSUBACK    = 11,
    PINGREQ     = 12,
    PINGRESP    = 13,
    DISCONNECT  = 14
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

```
<hr>
The first 2 `#define` refers to fixed sizes of the MQTT Fixed Header and of
every type of MQTT ACK packets, set for convenience, we'll use those later.

As shown, we leverage the **union**, a value that may have any of several
representations withing the same position in memory, to represent a byte. In
other words, inside unions, in contrast to normal **struct**, there can be only
one field with a value. Their position in memory are shared, this way using
**bitfields** we can effectively manipulate single bits or portions of a byte.

The first Control Packet we're going to define is the CONNECT. It' s the first
packet that must be sent when a client establish a new connection and it must
be exactly one, more than one CONNECT per client must be treated as a
violation of the protocol and the client must be dropped.<br>
For each CONNECT, a CONNACK packet must be sent in response.

<hr>
**src/mqtt.h**
<hr>
```c

struct mqtt_connect {
    union mqtt_header header;
    union {
        unsigned char byte;
        struct {
            int reserved : 1;
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
            unsigned reserved : 7;
        } bits;
    };
    unsigned char rc;
};

```
<hr>
From now on, the definition of other packets are trivial by reproducing the
pattern, accordingly to the documentation of MQTT v3.1.1.

We proceed with SUBSCRIBE, UNSUBSCRIBE and PUBLISH. SUBSCRIBE is the
only packet with a dedicated packet definition SUBACK, the other can be
defined as generic ACK, and type-named using **typedef** for semantic
separation.

<hr>
**src/mqtt.h**
<hr>

```c

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

```
<hr>

The remaining ACK packets, namely:
- PUBACK
- PUBREC
- PUBREL
- PUBCOMP
- UNSUBACK
- PINGREQ
- PINGRESP
- DISCONNECT

can be obtained by typedef'ing `struct ack`, just for semantic separation of
concerns. The last one, DISCONNECT, is not really an ACK but the format is
the same.

<hr>
**src/mqtt.h**
<hr>

```c

typedef struct mqtt_ack mqtt_puback;
typedef struct mqtt_ack mqtt_pubrec;
typedef struct mqtt_ack mqtt_pubrel;
typedef struct mqtt_ack mqtt_pubcomp;
typedef struct mqtt_ack mqtt_unsuback;
typedef union mqtt_header mqtt_pingreq;
typedef union mqtt_header mqtt_pingresp;
typedef union mqtt_header mqtt_disconnect;

```
<hr>

We can finally define a generic MQTT packet as a `union` of the previously
defined packets.

<hr>
**src/mqtt.h**
<hr>

```c

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

```
<hr>

We proceed now with the definition of some public functions, here in the header
we want to collect only those functions and structure that should be used by
other modules.

To handle the communication using the MQTT protocol we need essentially 4 functions,
2 for each direction of the interaction between server and client:

- A packing function (serializing or marshaling, I won't dive here in a
  dissertation on the correct usage of these terms)
- An unpacking function (deserializing/unmarshaling)

Supported by 2 functions to handle the encoding and decoding of the
Remaining Length in the Fixed Header part.

<hr>
**src/mqtt.h**
<hr>

```c
int mqtt_encode_length(unsigned char *, size_t);
unsigned long long mqtt_decode_length(const unsigned char **);
int unpack_mqtt_packet(const unsigned char *, union mqtt_packet *);
unsigned char *pack_mqtt_packet(const union mqtt_packet *, unsigned);
```
<hr>

We also add some utility functions to build packets and to release
heap-alloc'ed ones, nothing special here.

<hr>
**src/mqtt.h**
<hr>

```c

union mqtt_header *mqtt_packet_header(unsigned char);
struct mqtt_ack *mqtt_packet_ack(unsigned char , unsigned short);
struct mqtt_connack *mqtt_packet_connack(unsigned char, unsigned char, unsigned char);
struct mqtt_suback *mqtt_packet_suback(unsigned char, unsigned short,
                                       unsigned char *, unsigned short);
struct mqtt_publish *mqtt_packet_publish(unsigned char, unsigned short, size_t,
                                         unsigned char *, size_t, unsigned char *);
void mqtt_packet_release(union mqtt_packet *, unsigned);

#endif

```
<hr>

Fine. We have a decent header module that define all that we need for handling
the communication using the protocol. Let's now implement those functions.
First of all we define some "private" helpers, to pack and unpack each MQTT
packet, these will be called by the previously defined "public" functions
`unpack_mqtt_packet` and `pack_mqtt_packet`.

<hr>
**src/mqtt.c**
<hr>

```c

#include <stdlib.h>
#include <string.h>
#include "mqtt.h"

static size_t unpack_mqtt_connect(const unsigned char *, union mqtt_header *, union mqtt_packet *);
static size_t unpack_mqtt_publish(const unsigned char *, union mqtt_header *, union mqtt_packet *);
static size_t unpack_mqtt_subscribe(const unsigned char *, union mqtt_header *, union mqtt_packet *);
static size_t unpack_mqtt_unsubscribe(const unsigned char *, union mqtt_header *, union mqtt_packet *);
static size_t unpack_mqtt_ack(const unsigned char *, union mqtt_header *, union mqtt_packet *);
static unsigned char *pack_mqtt_header(const union mqtt_header *);
static unsigned char *pack_mqtt_ack(const union mqtt_packet *);
static unsigned char *pack_mqtt_connack(const union mqtt_packet *);
static unsigned char *pack_mqtt_suback(const union mqtt_packet *);
static unsigned char *pack_mqtt_publish(const union mqtt_packet *);

```
<hr>

### Packing and unpacking

Before continuing with the implementation of all defined functions on
`src/mqtt.h`, we need to implement some helpers functions to ease the pack and
unpack process of each received packet and also ready for send forged MQTT
packet.

Let's go fast here, it's just simple serialization/deserialization respecting
the network byte order (endianness, usually network byte order refers to
Big-endian order, while the majority of machines follow Little-endian
convention) of the packets.

<hr>
**src/pack.h**
<hr>

```c

#ifndef PACK_H
#define PACK_H

#include <stdio.h>
#include <stdint.h>

/* Reading data on const uint8_t pointer */
// bytes -> uint8_t
uint8_t unpack_u8(const uint8_t **);
// bytes -> uint16_t
uint16_t unpack_u16(const uint8_t **);
// bytes -> uint32_t
uint32_t unpack_u32(const uint8_t **);
// read a defined len of bytes
uint8_t *unpack_bytes(const uint8_t **, size_t, uint8_t *);
// Unpack a string prefixed by its length as a uint16 value
uint16_t unpack_string16(uint8_t **buf, uint8_t **dest)
/* Write data on const uint8_t pointer */
// append a uint8_t -> bytes into the bytestring
void pack_u8(uint8_t **, uint8_t);
// append a uint16_t -> bytes into the bytestring
void pack_u16(uint8_t **, uint16_t);
// append a uint32_t -> bytes into the bytestring
void pack_u32(uint8_t **, uint32_t);
// append len bytes into the bytestring
void pack_bytes(uint8_t **, uint8_t *);

#endif

```
<hr>

And the corresponding implementation

<hr>
**src/pack.c**
<hr>

```c

#include <string.h>
#include <stdlib.h>
#include <arpa/inet.h>
#include "pack.h"

// Reading data
uint8_t unpack_u8(const uint8_t **buf) {
    uint8_t val = **buf;
    (*buf)++;
    return val;
}

uint16_t unpack_u16(const uint8_t **buf) {
    uint16_t val;
    memcpy(&val, *buf, sizeof(uint16_t));
    (*buf) += sizeof(uint16_t);
    return ntohs(val);
}

uint32_t unpack_u32(const uint8_t **buf) {
    uint32_t val;
    memcpy(&val, *buf, sizeof(uint32_t));
    (*buf) += sizeof(uint32_t);
    return ntohl(val);
}

uint8_t *unpack_bytes(const uint8_t **buf, size_t len, uint8_t *str) {
    memcpy(str, *buf, len);
    str[len] = '\0';
    (*buf) += len;
    return str;
}

uint16_t unpack_string16(uint8_t **buf, uint8_t **dest) {
    uint16_t len = unpack_u16(buf);
    *dest = malloc(len + 1);
    *dest = unpack_bytes(buf, len, *dest);
    return len;
}

// Write data
void pack_u8(uint8_t **buf, uint8_t val) {
    **buf = val;
    (*buf) += sizeof(uint8_t);
}

void pack_u16(uint8_t **buf, uint16_t val) {
    uint16_t htonsval = htons(val);
    memcpy(*buf, &htonsval, sizeof(uint16_t));
    (*buf) += sizeof(uint16_t);
}

void pack_u32(uint8_t **buf, uint32_t val) {
    uint32_t htonlval = htonl(val);
    memcpy(*buf, &htonlval, sizeof(uint32_t));
    (*buf) += sizeof(uint32_t);
}

void pack_bytes(uint8_t **buf, uint8_t *str) {
    size_t len = strlen((char *) str);
    memcpy(*buf, str, len);
    (*buf) += len;
}

```
<hr>

This allow us to handle incoming stream of bytes and forge them to respond to
connected clients. Let's move one.

### Back to mqtt module

After the creation of `pack` module we should include it into the `mqtt` source:

<hr>
**src/mqtt.c**
<hr>

```c

#include "pack.h"

```
<hr>

The first step will be the implementation of the Fixed Header Remaining Length
functions. The MQTT documentation suggests a pseudo-code implementation in one
of the first paragraphs, we'll stick to that, it's quiet simple and clear.
We'll see why and how after the first byte of the Fixed Header, the next 1 or
2 or 3 or 4 bytes are used to encode the remaining bytes of the packet.

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

<hr>
**src/mqtt.c**
<hr>

```c

/*
 * MQTT v3.1.1 standard, Remaining length field on the fixed header can be at
 * most 4 bytes.
 */
static const int MAX_LEN_BYTES = 4;

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
        short d = len % 128;
        len /= 128;
        /* if there are more digits to encode, set the top bit of this digit */
        if (len > 0)
            d |= 128;
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

```
<hr>

Now we can read the first header byte and the total length of the packet. Let's
move on with the unpacking of the CONNECT packet.
It's the packet with more flags and the second one in length behind only the
PUBLISH packet.

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

To clarify the concept, let's suppose we receive a CONNECT packet with:

- username and password flag to 1
- username = "hello"
- password = "nacho"
- client ID = "danzan"


| Field  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;              | size (bytes) &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;| offset (byte position) &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  | Description                   |
|----------------------|:-----------:|:-----------------------:|---------------------------------------------------------------------|
| Packet type          |  1 (4 bits) |   0                     | Connect type                                                        |
| Length               |    1        |   1                     | 32 bytes length, being it < 127 bytes, it requires only 1 byte      |
| Protocol name length |    2        |   2                     | 4 bytes length                                                      |
| Protocol name (MQTT) |    4        |   4                     | 'M' 'Q' 'T' 'T'                                                     |
| Protocol level       |    1        |   8                     | For version 3.1.1 the level is 4                                    |
| Connect flags        |    1        |   9                     | Username, password, will retain, will QoS, will flag, clean session |
| Keepalive            |    2        |   10                    | 16-bit word, maximum value is 18 hr 12 min 15 seconds               |
| Client ID length     |    2        |   12                    | 2 bytes, 6 is the length of the Client ID (danzan)                  |
| Client ID            |    6        |   14                    | 'd' 'a' 'n' 'z' 'a' 'n'                                             |
| Username length      |    2        |   20                    | 2 bytes, 5 is the length of the username (hello)                    |
| Username             |    5        |   22                    | 'h' 'e' 'l' 'l' 'o'                                                 |
| Password length      |    2        |   27                    | 2 bytes, 5 is the length of the password (nacho)                    |
| Password             |    5        |   29                    | 'n' 'a' 'c' 'h' 'o'                                                 |

<br>
Having will flags to 0 there's no need to decode those fields, as they don't
even appear. We have as a result a total length 34 bytes packet including fixed
header.

<hr>
**src/mqtt.c**
<hr>

```c

/*
 * MQTT unpacking functions
 */

static size_t unpack_mqtt_connect(const unsigned char *buf,
                                  union mqtt_header *hdr,
                                  union mqtt_packet *pkt) {
    struct mqtt_connect connect = { .header = *hdr };
    pkt->connect = connect;
    const unsigned char *init = buf;
    /*
     * Second byte of the fixed header, contains the length of remaining bytes
     * of the connect packet
     */
    size_t len = mqtt_decode_length(&buf);
    /*
     * For now we ignore checks on protocol name and reserved bits, just skip
     * to the 8th byte
     */
    buf = init + 8;
    /* Read variable header byte flags */
    pkt->connect.byte = unpack_u8((const uint8_t **) &buf);
    /* Read keepalive MSB and LSB (2 bytes word) */
    pkt->connect.payload.keepalive = unpack_u16((const uint8_t **) &buf);
    /* Read CID length (2 bytes word) */
    uint16_t cid_len = unpack_u16((const uint8_t **) &buf);
    /* Read the client id */
    if (cid_len > 0) {
        pkt->connect.payload.client_id = malloc(cid_len + 1);
        unpack_bytes((const uint8_t **) &buf, cid_len,
                     pkt->connect.payload.client_id);
    }
    /* Read the will topic and message if will is set on flags */
    if (pkt->connect.bits.will == 1) {
        unpack_string16(&buf, &pkt->connect.payload.will_topic);
        unpack_string16(&buf, &pkt->connect.payload.will_message);
    }
    /* Read the username if username flag is set */
    if (pkt->connect.bits.username == 1)
        unpack_string16(&buf, &pkt->connect.payload.username);
    /* Read the password if password flag is set */
    if (pkt->connect.bits.password == 1)
        unpack_string16(&buf, &pkt->connect.payload.password);
    return len;
}

```
<hr>

The PUBLISH packet now:

```markdown

 |   Bit    |  7  |  6  |  5  |  4  |  3  |  2  |  1  |   0    |  <-- Fixed Header
 |----------|-----------------------|--------------------------|
 | Byte 1   |      MQTT type 3      | dup |    QoS    | retain |
 |----------|--------------------------------------------------|
 | Byte 2   |                                                  |
 |  .       |               Remaining Length                   |
 |  .       |                                                  |
 | Byte 5   |                                                  |
 |----------|--------------------------------------------------|  <-- Variable Header
 | Byte 6   |                Topic len MSB                     |
 | Byte 7   |                Topic len LSB                     |
 |-------------------------------------------------------------|
 | Byte 8   |                                                  |
 |   .      |                Topic name                        |
 | Byte N   |                                                  |
 |----------|--------------------------------------------------|
 | Byte N+1 |            Packet Identifier MSB                 |
 | Byte N+2 |            Packet Identifier LSB                 |
 |----------|--------------------------------------------------|  <-- Payload
 | Byte N+3 |                   Payload                        |
 | Byte N+M |                                                  |

```


Packet identifier MSB and LSB are present in the packet if and only if the QoS
level is > 0, with a QoS set to *at most once* there's no need for a packet ID.
Payload length is calculated by subtracting the Remaining Length with all the
other fields already unpacked.

<hr>
**src/mqtt.c**
<hr>

```c

static size_t unpack_mqtt_publish(const unsigned char *buf,
                                  union mqtt_header *hdr,
                                  union mqtt_packet *pkt) {
    struct mqtt_publish publish = { .header = *hdr };
    pkt->publish = publish;
    /*
     * Second byte of the fixed header, contains the length of remaining bytes
     * of the connect packet
     */
    size_t len = mqtt_decode_length(&buf);
    /* Read topic length and topic of the soon-to-be-published message */
    pkt->publish.topiclen = unpack_string16(&buf, &pkt->publish.topic);
    uint16_t message_len = len;
    /* Read packet id */
    if (publish.header.bits.qos > AT_MOST_ONCE) {
        pkt->publish.pkt_id = unpack_u16((const uint8_t **) &buf);
        message_len -= sizeof(uint16_t);
    }
    /*
     * Message len is calculated subtracting the length of the variable header
     * from the Remaining Length field that is in the Fixed Header
     */
    message_len -= (sizeof(uint16_t) + topic_len);
    pkt->publish.payloadlen = message_len;
    pkt->publish.payload = malloc(message_len + 1);
    unpack_bytes((const uint8_t **) &buf, message_len, pkt->publish.payload);
    return len;
}

```
<hr>

Subscribe and unsubscribe packets are fairly similar, they reflect the
PUBLISH packet, but for payload they have a list of tuple consisting in a
pair (topic, QoS). Their implementation is practically identical, with the
only difference in the payload part, where UNSUBSCRIBE doesn't specify QoS
for each topic.

<hr>
**src/mqtt.c**
<hr>

```c

static size_t unpack_mqtt_subscribe(const unsigned char *buf,
                                    union mqtt_header *hdr,
                                    union mqtt_packet *pkt) {
    struct mqtt_subscribe subscribe = { .header = *hdr };
    /*
     * Second byte of the fixed header, contains the length of remaining bytes
     * of the connect packet
     */
    size_t len = mqtt_decode_length(&buf);
    size_t remaining_bytes = len;
    /* Read packet id */
    subscribe.pkt_id = unpack_u16((const uint8_t **) &buf);
    remaining_bytes -= sizeof(uint16_t);
    /*
     * Read in a loop all remaining bytes specified by len of the Fixed Header.
     * From now on the payload consists of 3-tuples formed by:
     *  - topic length
     *  - topic filter (string)
     *  - qos
     */
    int i = 0;
    while (remaining_bytes > 0) {
        /* Read length bytes of the first topic filter */
        remaining_bytes -= sizeof(uint16_t);
        /* We have to make room for additional incoming tuples */
        subscribe.tuples = realloc(subscribe.tuples,
                                   (i+1) * sizeof(*subscribe.tuples));
        subscribe.tuples[i].topic_len =
            unpack_string16(&buf, &subscribe.tuples[i].topic);
        remaining_bytes -= subscribe.tuples[i].topic_len;
        subscribe.tuples[i].qos = unpack_u8((const uint8_t **) &buf);
        len -= sizeof(uint8_t);
        i++;
    }
    subscribe.tuples_len = i;
    pkt->subscribe = subscribe;
    return len;
}

static size_t unpack_mqtt_unsubscribe(const unsigned char *buf,
                                      union mqtt_header *hdr,
                                      union mqtt_packet *pkt) {
    struct mqtt_unsubscribe unsubscribe = { .header = *hdr };
    /*
     * Second byte of the fixed header, contains the length of remaining bytes
     * of the connect packet
     */
    size_t len = mqtt_decode_length(&buf);
    size_t remaining_bytes = len;
    /* Read packet id */
    unsubscribe.pkt_id = unpack_u16((const uint8_t **) &buf);
    remaining_bytes -= sizeof(uint16_t);
    /*
     * Read in a loop all remaining bytes specified by len of the Fixed Header.
     * From now on the payload consists of 2-tuples formed by:
     *  - topic length
     *  - topic filter (string)
     */
    int i = 0;
    while (remaining_bytes > 0) {
        /* Read length bytes of the first topic filter */
        remaining_bytes -= sizeof(uint16_t);
        /* We have to make room for additional incoming tuples */
        unsubscribe.tuples = realloc(unsubscribe.tuples,
                                     (i+1) * sizeof(*unsubscribe.tuples));
        unsubscribe.tuples[i].topic_len =
            unpack_string16(&buf, &unsubscribe.tuples[i].topic);
        remaining_bytes -= unsubscribe.tuples[i].topic_len;
        i++;
    }
    unsubscribe.tuples_len = i;
    pkt->unsubscribe = unsubscribe;
    return len;
}

```
<hr>

And finally the ACK. In MQTT doesn't exists generic acks, but the structure
of most of the ack-like packets is practically the same for each one, formed by
a header and a packet ID.

These are:

- PUBACK
- PUBREC
- PUBREL
- PUBCOMP
- UNSUBACK

<hr>
**src/mqtt.c**
<hr>

```c

static size_t unpack_mqtt_ack(const unsigned char *buf,
                              union mqtt_header *hdr,
                              union mqtt_packet *pkt) {
    struct mqtt_ack ack = { .header = *hdr };
    /*
     * Second byte of the fixed header, contains the length of remaining bytes
     * of the connect packet
     */
    size_t len = mqtt_decode_length(&buf);
    ack.pkt_id = unpack_u16((const uint8_t **) &buf);
    pkt->ack = ack;
    return len;
}

```
<hr>

We have now all needed helpers functions to implement our only exposed function
on the header `mqtt_mqtt_packet`. It ended up being a fairly short and simple
function, we'll use a static array to map all helper functions, making it O(1)
the selection of the correct unpack function based on the Control Packet type.
To be noted that in case of a disconnect, a pingreq or pingresp packet we only
need a single byte, with remaining length 0.

<hr>
**src/mqtt.c**
<hr>

```c

typedef size_t mqtt_unpack_handler(const unsigned char *,
                                   union mqtt_header *,
                                   union mqtt_packet *);
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

int unpack_mqtt_packet(const unsigned char *buf, union mqtt_packet *pkt) {
    int rc = 0;
    /* Read first byte of the fixed header */
    unsigned char type = *buf;
    union mqtt_header header = {
        .byte = type
    };
    if (header.bits.type == DISCONNECT
        || header.bits.type == PINGREQ
        || header.bits.type == PINGRESP)
        pkt->header = header;
    else
        /* Call the appropriate unpack handler based on the message type */
        rc = unpack_handlers[header.bits.type](++buf, &header, pkt);
    return rc;
}

```
<hr>

The first part ends here, at this point we have two modules, one of utility for
general serialization operations and one to handle the protocol itself accordingly
to the standard defined by OASIS.

```bash
sol/
 ├── src/
 │    ├── mqtt.h
 │    ├── mqtt.c
 │    ├── pack.h
 │    └── pack.c
 ├── CHANGELOG
 ├── CMakeLists.txt
 ├── COPYING
 └── README.md
```

Just `git commit` and `git push`. Cya.

[Part-2](../sol-mqtt-broker-p2) will deal with the
networking utilities needed to setup our communication layer and thus the server.
