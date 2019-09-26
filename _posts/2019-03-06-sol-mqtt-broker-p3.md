---
layout: post
title: "Sol - An MQTT broker from scratch. Part 3 - Server"
description: "Writing an MQTT broker from scratch, to really understand something you have to build it."
categories: c unix tutorial
---

This part deal with the implementation of the server part of our application, by
using the `network` module we drafted on [part-2]({{site.url}}{{site.baseurl}}/2019/03/04/sol-mqtt-broker-p2)
it should be relative easy to handle incoming commands from a MQTT clients respecting 3.1.1
standards as we defined on [part 1]({{site.url}}{{site.baseurl}}/2019/03/03/sol-mqtt-broker).<br>
<!--more-->
Our header file will be extremely simple, the only function we want to make
accessible from the outside will be a transparent `start_server`, accepting
only two trivial arguments:

- an IP address
- a port to listen on

We will define also two constants for the **epoll** interface creation, the
number of events we want to monitor concurrently and the timeout, two
properties which could be easily moved to a configuration module for further
improvements.

**src/server.h**

{% highlight c %}

#ifndef SERVER_H
#define SERVER_H

/*
 * Epoll default settings for concurrent events monitored and timeout, -1
 * means no timeout at all, blocking undefinitely
 */
#define EPOLL_MAX_EVENTS    256
#define EPOLL_TIMEOUT       -1

/* Error codes for packet reception, signaling respectively
 * - client disconnection
 * - error reading packet
 * - error packet sent exceeds size defined by configuration (generally default
 *   to 2MB)
 */
#define ERRCLIENTDC         1
#define ERRPACKETERR        2
#define ERRMAXREQSIZE       3

/* Return code of handler functions, signaling if there's data payload to be
 * sent out or if the server just need to re-arm closure for reading incoming
 * bytes
 */
#define REARM_R             0
#define REARM_W             1

int start_server(const char *, const char *);

#endif

{% endhighlight %}

The implementation part will be a little bigger than it seems, all our handler
functions and callbacks will be contained here for now, so let's start with
the 3 basic callbacks that every server will need to effectively communicate with
clients after it started listening:

- an accept callback
- a read callback for read events
- a write callback to send out data

We will also forward declare some functions, mostly handlers and a mapping much
like the one used to pack and unpack payloads on mqtt module, we took them as is,
to make thing easier for the near-future development.

**src/server.c**

{% highlight c %}

#define _POSIX_C_SOURCE 200809L
#include <time.h>
#include <errno.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include "pack.h"
#include "util.h"
#include "mqtt.h"
#include "core.h"
#include "network.h"
#include "hashtable.h"
#include "config.h"
#include "server.h"


/* Seconds in a Sol, easter egg */
static const double SOL_SECONDS = 88775.24;

/*
 * General informations of the broker, all fields will be published
 * periodically to internal topics
 */
static struct sol_info info;

/* Broker global instance, contains the topic trie and the clients hashtable */
static struct sol sol;

/*
 * Prototype for a command handler, it accepts a pointer to the closure as the
 * link to the client sender of the command and a pointer to the packet itself
 */
typedef int handler(struct closure *, union mqtt_packet *);

/* Command handler, each one have responsibility over a defined command packet */
static int connect_handler(struct closure *, union mqtt_packet *);
static int disconnect_handler(struct closure *, union mqtt_packet *);
static int subscribe_handler(struct closure *, union mqtt_packet *);
static int unsubscribe_handler(struct closure *, union mqtt_packet *);
static int publish_handler(struct closure *, union mqtt_packet *);
static int puback_handler(struct closure *, union mqtt_packet *);
static int pubrec_handler(struct closure *, union mqtt_packet *);
static int pubrel_handler(struct closure *, union mqtt_packet *);
static int pubcomp_handler(struct closure *, union mqtt_packet *);
static int pingreq_handler(struct closure *, union mqtt_packet *);

/* Command handler mapped usign their position paired with their type */
static handler *handlers[15] = {
    NULL,
    connect_handler,
    NULL,
    publish_handler,
    puback_handler,
    pubrec_handler,
    pubrel_handler,
    pubcomp_handler,
    subscribe_handler,
    NULL,
    unsubscribe_handler,
    NULL,
    pingreq_handler,
    NULL,
    disconnect_handler
};

/*
 * Connection structure for private use of the module, mainly for accepting
 * new connections
 */
struct connection {
    char ip[INET_ADDRSTRLEN + 1];
    int fd;
};

/* I/O closures, for the 3 main operation of the server
 * - Accept a new connecting client
 * - Read incoming bytes from connected clients
 * - Write output bytes to connected clients
 */
static void on_read(struct evloop *, void *);

static void on_write(struct evloop *, void *);

static void on_accept(struct evloop *, void *);

/*
 * Periodic task callback, will be executed every N seconds defined on the
 * configuration
 */
static void publish_stats(struct evloop *, void *);

/*
 * Accept a new incoming connection assigning ip address and socket descriptor
 * to the connection structure pointer passed as argument
 */
static int accept_new_client(int fd, struct connection *conn) {

    if (!conn)
        return -1;

    /* Accept the connection */
    int clientsock = accept_connection(fd);

    /* Abort if not accepted */
    if (clientsock == -1)
        return -1;

    /* Just some informations retrieval of the new accepted client connection */
    struct sockaddr_in addr;
    socklen_t addrlen = sizeof(addr);

    if (getpeername(clientsock, (struct sockaddr *) &addr, &addrlen) < 0)
        return -1;

    char ip_buff[INET_ADDRSTRLEN + 1];
    if (inet_ntop(AF_INET, &addr.sin_addr, ip_buff, sizeof(ip_buff)) == NULL)
        return -1;

    struct sockaddr_in sin;
    socklen_t sinlen = sizeof(sin);

    if (getsockname(fd, (struct sockaddr *) &sin, &sinlen) < 0)
        return -1;

    conn->fd = clientsock;
    strcpy(conn->ip, ip_buff);

    return 0;
}

/*
 * Handle new connection, create a a fresh new struct client structure and link
 * it to the fd, ready to be set in EPOLLIN event
 */
static void on_accept(struct evloop *loop, void *arg) {

    /* struct connection *server_conn = arg; */
    struct closure *server = arg;
    struct connection conn;

    accept_new_client(server->fd, &conn);

    /* Create a client structure to handle his context connection */
    struct closure *client_closure = malloc(sizeof(*client_closure));
    if (!client_closure)
        return;

    /* Populate client structure */
    client_closure->fd = conn.fd;
    client_closure->obj = NULL;
    client_closure->payload = NULL;
    client_closure->args = client_closure;
    client_closure->call = on_read;
    generate_uuid(client_closure->closure_id);

    hashtable_put(sol.closures, client_closure->closure_id, client_closure);

    /* Add it to the epoll loop */
    evloop_add_callback(loop, client_closure);

    /* Rearm server fd to accept new connections */
    evloop_rearm_callback_read(loop, server);

    /* Record the new client connected */
    info.nclients++;
    info.nconnections++;

    sol_info("New connection from %s on port %s", conn.ip, conf->port);
}

{% endhighlight %}

As you can see, I defined two static functions (in C, while not strictly a
correct use of the term, static functions can only be seen in the scope of
the module of definition, almost like a private method on a class in OOP
programming), the `accept_new_client` which by using functions from `network`
module to accept new connecting clients and the `on_accept` function, which
will be the effective callback for accepting new connections.

The `accept_new_client` function expects a struct `connection` that i added
for convenience and for reuse of old code from another codebase of mine,
not strictly necessary to follow this pattern.

**src/server.c**

{% highlight c %}

/*
 * Parse packet header, it is required at least the Fixed Header of each
 * packed, which is contained in the first 2 bytes in order to read packet
 * type and total length that we need to recv to complete the packet.
 *
 * This function accept a socket fd, a buffer to read incoming streams of
 * bytes and a structure formed by 2 fields:
 *
 * - buf -> a byte buffer, it will be malloc'ed in the function and it will
 *          contain the serialized bytes of the incoming packet
 * - flags -> flags pointer, copy the flag setting of the incoming packet,
 *            again for simplicity and convenience of the caller.
 */
static ssize_t recv_packet(int clientfd, unsigned char *buf, char *command) {

    ssize_t nbytes = 0;

    /* Read the first byte, it should contain the message type code */
    if ((nbytes = recv_bytes(clientfd, buf, 1)) <= 0)
        return -ERRCLIENTDC;

    unsigned char byte = *buf;
    buf++;

    if (DISCONNECT < byte || CONNECT > byte)
        return -ERRPACKETERR;

    /*
     * Read remaning length bytes which starts at byte 2 and can be long to 4
     * bytes based on the size stored, so byte 2-5 is dedicated to the packet
     * length.
     */
    unsigned char buff[4];
    int count = 0;
    int n = 0;
    do {
        if ((n = recv_bytes(clientfd, buf+count, 1)) <= 0)
            return -ERRCLIENTDC;
        buff[count] = buf[count];
        nbytes += n;
    } while (buff[count++] & (1 << 7));

    // Reset temporary buffer
    const unsigned char *pbuf = &buff[0];
    unsigned long long tlen = mqtt_decode_length(&pbuf);

    /*
     * Set return code to -ERRMAXREQSIZE in case the total packet len exceeds
     * the configuration limit `max_request_size`
     */
    if (tlen > conf->max_request_size) {
        nbytes = -ERRMAXREQSIZE;
        goto exit;
    }

    /* Read remaining bytes to complete the packet */
    if ((n = recv_bytes(clientfd, buf + 1, tlen)) < 0)
        goto err;

    nbytes += n;

    *command = byte;

exit:

    return nbytes;

err:

    shutdown(clientfd, 0);
    close(clientfd);

    return nbytes;

}

/* Handle incoming requests, after being accepted or after a reply */
static void on_read(struct evloop *loop, void *arg) {

    struct closure *cb = arg;

    /* Raw bytes buffer to handle input from client */
    unsigned char *buffer = malloc(conf->max_request_size);

    ssize_t bytes = 0;
    char command = 0;

    /*
     * We must read all incoming bytes till an entire packet is received. This
     * is achieved by following the MQTT v3.1.1 protocol specifications, which
     * send the size of the remaining packet as the second byte. By knowing it
     * we know if the packet is ready to be deserialized and used.
     */
    bytes = recv_packet(cb->fd, buffer, &command);

    /*
     * Looks like we got a client disconnection.
     *
     * TODO: Set a error_handler for ERRMAXREQSIZE instead of dropping client
     *       connection, explicitly returning an informative error code to the
     *       client connected.
     */
    if (bytes == -ERRCLIENTDC || bytes == -ERRMAXREQSIZE)
        goto exit;

    /*
     * If a not correct packet received, we must free the buffer and reset the
     * handler to the request again, setting EPOLL to EPOLLIN
     */
    if (bytes == -ERRPACKETERR)
        goto errdc;

    info.bytes_recv++;

    /*
     * Unpack received bytes into a mqtt_packet structure and execute the
     * correct handler based on the type of the operation.
     */
    union mqtt_packet packet;
    unpack_mqtt_packet(buffer, &packet);

    union mqtt_header hdr = { .byte = command };

    /* Execute command callback */
    int rc = handlers[hdr.bits.type](cb, &packet);

    if (rc == REARM_W) {

        cb->call = on_write;

        /*
         * Reset handler to read_handler in order to read new incoming data and
         * EPOLL event for read fds
         */
        evloop_rearm_callback_write(loop, cb);
    } else if (rc == REARM_R) {
        cb->call = on_read;
        evloop_rearm_callback_read(loop, cb);
    }

    // Disconnect packet received

exit:

    free(buffer);

    return;

errdc:

    free(buffer);

    sol_error("Dropping client");
    shutdown(cb->fd, 0);
    close(cb->fd);

    hashtable_del(sol.clients, ((struct sol_client *) cb->obj)->client_id);
    hashtable_del(sol.closures, cb->closure_id);

    info.nclients--;

    info.nconnections--;

    return;
}

static void on_write(struct evloop *loop, void *arg) {

    struct closure *cb = arg;

    ssize_t sent;
    if ((sent = send_bytes(cb->fd, cb->payload->data, cb->payload->size)) < 0)
        sol_error("Error writing on socket to client %s: %s",
                  ((struct sol_client *) cb->obj)->client_id, strerror(errno));

    // Update information stats
    info.bytes_sent += sent;
    bytestring_release(cb->payload);
    cb->payload = NULL;

    /*
     * Re-arm callback by setting EPOLL event on EPOLLIN to read fds and
     * re-assigning the callback `on_read` for the next event
     */
    cb->call = on_read;
    evloop_rearm_callback_read(loop, cb);
}

{% endhighlight %}

Another 3 static functions added, as shown, there's a `recv_packet` function
that like foretold by his name, have the duty of receiving streams of bytes till
a full MQTT packet is built, relying on functions from `mqtt` module, the other
two are respectively `on_read` and `on_write`.

To be noted that, these 3 callbacks, rearm the socket using previously defined
helper bouncing the ball on each others. `on_read` callback specifically, based
upon a return code from the calling of a handler (chosen in turn to the type of
command received from the client on the line
`int rc = handlers[hdr.bits.type](cb, &packet)`), rearm the socket for read or
write or not at all. This last case will cover disconnections for errors and
legitimate DISCONNECT packets.

On the write callback we see that the `send_bytes` call pass in a payload with
data and size as fields; it's a convenient structure I defined to make it simpler
to track number of bytes on an array, this case to know how many bytes to write out.
We have to re-open the `src/pack.h` and `src/pack.c` to add these utilities.

**src/pack.h**

{% highlight c %}

/*
 * bytestring structure, provides a convenient way of handling byte string data.
 * It is essentially an unsigned char pointer that track the position of the
 * last written byte and the total size of the bystestring
 */
struct bytestring {
    size_t size;
    size_t last;
    unsigned char *data;
};

/*
 * const struct bytestring constructor, it require a size cause we use a bounded
 * bytestring, e.g. no resize over a defined size
 */
struct bytestring *bytestring_create(size_t);
void bytestring_init(struct bytestring *, size_t);
void bytestring_release(struct bytestring *);
void bytestring_reset(struct bytestring *);

{% endhighlight %}

And their trivial implementation

**src/pack.c**

{% highlight c %}

struct bytestring *bytestring_create(size_t len) {
    struct bytestring *bstring = malloc(sizeof(*bstring));
    bytestring_init(bstring, len);
    return bstring;
}

void bytestring_init(struct bytestring *bstring, size_t size) {
    if (!bstring)
        return;
    bstring->size = size;
    bstring->data = malloc(sizeof(unsigned char) * size);
    bytestring_reset(bstring);
}

void bytestring_release(struct bytestring *bstring) {
    if (!bstring)
        return;
    free(bstring->data);
    free(bstring);
}

void bytestring_reset(struct bytestring *bstring) {
    if (!bstring)
        return;
    bstring->last = 0;
    memset(bstring->data, 0, bstring->size);
}

{% endhighlight %}

## Generic utilities

Let's open a brief parenthesis, for my projects, generally there's always need
for generic helpers and utility functions, I usually collect them into a
dedicated `util` module; that's the case for call like `sol_info`, `sol_debug`
and `sol_error` on those chunks of code previously analysed.

Our logging requirements is so simple that there's no need for a dedicated module
yet, so I generally add those logging functions to the `util` module.

**src/util.h**

{% highlight c %}

#ifndef UTIL_H
#define UTIL_H

#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <strings.h>

#define UUID_LEN     37
#define MAX_LOG_SIZE 119

enum log_level { DEBUG, INFORMATION, WARNING, ERROR };

int number_len(size_t);
int parse_int(const char *);
int generate_uuid(char *);
char *remove_occur(char *, char) ;
char *append_string(char *, char *, size_t);

/* Logging */
void sol_log_init(const char *);
void sol_log_close(void);
void sol_log(int, const char *, ...);

#define log(...) sol_log( __VA_ARGS__ )
#define sol_debug(...) log(DEBUG, __VA_ARGS__)
#define sol_warning(...) log(WARNING, __VA_ARGS__)
#define sol_error(...) log(ERROR, __VA_ARGS__)
#define sol_info(...) log(INFORMATION, __VA_ARGS__)

#define STREQ(s1, s2, len) strncasecmp(s1, s2, len) == 0 ? true : false

#endif

{% endhighlight %}

And here we go, a log function with some macros to conveniently call the
correct level of logging, with an additional utility macro `STREQ` to compare two
strings.

**src/util.c**

{% highlight c %}

#include <time.h>
#include <ctype.h>
#include <errno.h>
#include <assert.h>
#include <string.h>
#include <stdlib.h>
#include <stdarg.h>
#include <uuid/uuid.h>
#include "util.h"
#include "config.h"

static FILE *fh = NULL;

void sol_log_init(const char *file) {
    assert(file);
    fh = fopen(file, "a+");
    if (!fh)
        printf("%lu * WARNING: Unable to open file %s\n",
               (unsigned long) time(NULL), file);
}

void sol_log_close(void) {
    if (fh) {
        fflush(fh);
        fclose(fh);
    }
}

void sol_log(int level, const char *fmt, ...) {

    assert(fmt);

    va_list ap;
    char msg[MAX_LOG_SIZE + 4];

    if (level < conf->loglevel)
        return;

    va_start(ap, fmt);
    vsnprintf(msg, sizeof(msg), fmt, ap);
    va_end(ap);

    /* Truncate message too long and copy 3 bytes to make space for 3 dots */
    memcpy(msg + MAX_LOG_SIZE, "...", 3);
    msg[MAX_LOG_SIZE + 3] = '\0';

    // Distinguish message level prefix
    const char *mark = "#i*!";

    // Open two handler, one for standard output and a second for the
    // persistent log file
    FILE *fp = stdout;

    if (!fp)
        return;

    fprintf(fp, "%lu %c %s\n", (unsigned long) time(NULL), mark[level], msg);
    if (fh)
        fprintf(fh, "%lu %c %s\n", (unsigned long) time(NULL), mark[level], msg);

    fflush(fp);
    if (fh)
        fflush(fh);
}

/*
 * Return the 'length' of a positive number, as the number of chars it would
 * take in a string
 */
int number_len(size_t number) {
    int len = 1;
    while (number) {
        len++;
        number /= 10;
    }
    return len;
}

/* Parse the integer part of a string, by effectively iterate through it and
   converting the numbers found */
int parse_int(const char *string) {
    int n = 0;

    while (*string && isdigit(*string)) {
        n = (n * 10) + (*string - '0');
        string++;
    }
    return n;
}

char *remove_occur(char *str, char c) {
    char *p = str;
    char *pp = str;

    while (*p) {
        *pp = *p++;
        pp += (*pp != c);
    }

    *pp = '\0';

    return str;
}

/*
 * Append a string to another, the destination string must be NUL-terminated
 * and long enough to contain the resulting string, for the chunk part that
 * will be appended the function require the length, the resulting string will
 * be heap alloced and nul-terminated.
 */
char *append_string(char *src, char *chunk, size_t chunklen) {
    size_t srclen = strlen(src);
    char *ret = malloc(srclen + chunklen + 1);
    memcpy(ret, src, srclen);
    memcpy(ret + srclen, chunk, chunklen);
    ret[srclen + chunklen] = '\0';
    return ret;
}

int generate_uuid(char *uuid_placeholder) {

    /* Generate random uuid */
    uuid_t binuuid;
    uuid_generate_random(binuuid);
    uuid_unparse(binuuid, uuid_placeholder);

    return 0;
}

{% endhighlight %}

This simple functions allow us to have a pretty decent logging system, by
calling `sol_log_init` on the main function we can also persist logs on disk
by passing a path on the filesystem.

We finally arrive to write our `start_server` function, which uses all other
functions already defined. Basically it acts as an entry point, setting up all
global structures and the first closure for accepting incoming connections.

**src/server.c**

{% highlight c %}

/*
 * Statistics topics, published every N seconds defined by configuration
 * interval
 */
#define SYS_TOPICS 14

static const char *sys_topics[SYS_TOPICS] = {
    "$SOL/",
    "$SOL/broker/",
    "$SOL/broker/clients/",
    "$SOL/broker/bytes/",
    "$SOL/broker/messages/",
    "$SOL/broker/uptime/",
    "$SOL/broker/uptime/sol",
    "$SOL/broker/clients/connected/",
    "$SOL/broker/clients/disconnected/",
    "$SOL/broker/bytes/sent/",
    "$SOL/broker/bytes/received/",
    "$SOL/broker/messages/sent/",
    "$SOL/broker/messages/received/",
    "$SOL/broker/memory/used"
};

static void run(struct evloop *loop) {
    if (evloop_wait(loop) < 0) {
        sol_error("Event loop exited unexpectedly: %s", strerror(loop->status));
        evloop_free(loop);
    }
}

/*
 * Cleanup function to be passed in as destructor to the Hashtable for
 * connecting clients
 */
static int client_destructor(struct hashtable_entry *entry) {

    if (!entry)
        return -1;

    struct sol_client *client = entry->val;

    if (client->client_id)
        free(client->client_id);

    free(client);

    return 0;
}

/*
 * Cleanup function to be passed in as destructor to the Hashtable for
 * registered closures.
 */
static int closure_destructor(struct hashtable_entry *entry) {

    if (!entry)
        return -1;

    struct closure *closure = entry->val;

    if (closure->payload)
        bytestring_release(closure->payload);

    free(closure);

    return 0;
}

int start_server(const char *addr, const char *port) {

    /* Initialize global Sol instance */
    trie_init(&sol.topics);
    sol.clients = hashtable_create(client_destructor);
    sol.closures = hashtable_create(closure_destructor);

    struct closure server_closure;

    /* Initialize the sockets, first the server one */
    server_closure.fd = make_listen(addr, port, conf->socket_family);
    server_closure.payload = NULL;
    server_closure.args = &server_closure;
    server_closure.call = on_accept;
    generate_uuid(server_closure.closure_id);

    /* Generate stats topics */
    for (int i = 0; i < SYS_TOPICS; i++)
        sol_topic_put(&sol, topic_create(strdup(sys_topics[i])));

    struct evloop *event_loop = evloop_create(EPOLL_MAX_EVENTS, EPOLL_TIMEOUT);

    /* Set socket in EPOLLIN flag mode, ready to read data */
    evloop_add_callback(event_loop, &server_closure);

    /* Add periodic task for publishing stats on SYS topics */
    // TODO Implement
    struct closure sys_closure = {
        .fd = 0,
        .payload = NULL,
        .args = &sys_closure,
        .call = publish_stats
    };

    generate_uuid(sys_closure.closure_id);

    /* Schedule as periodic task to be executed every 5 seconds */
    evloop_add_periodic_task(event_loop, conf->stats_pub_interval,
                             0, &sys_closure);

    sol_info("Server start");
    info.start_time = time(NULL);

    run(event_loop);

    hashtable_release(sol.clients);
    hashtable_release(sol.closures);

    sol_info("Sol v%s exiting", VERSION);

    return 0;
}
{% endhighlight %}

Ok, we have now a (almost) fully functioning server that uses our toyish
callback system to handle traffic. Let's add some additional code to the server
header, like that `info` structure and a global structure named `sol` on the
source .c, we'll be back on that soon.

**src/server.h**

{% highlight c %}

/* Global informations statistics structure */
struct sol_info {
    /* Number of clients currently connected */
    int nclients;
    /* Total number of clients connected since the start */
    int nconnections;
    /* Timestamp of the start time */
    long long start_time;
    /* Total number of bytes received */
    long long bytes_recv;
    /* Total number of bytes sent out */
    long long bytes_sent;
    /* Total number of sent messages */
    long long messages_sent;
    /* Total number of received messages */
    long long messages_recv;
};

{% endhighlight %}

This is directly linked to the periodic task added in the start_server function

{% highlight c %}

/* Add periodic task for publishing stats on SYS topics */
// TODO Implement
struct closure sys_closure = {
    .fd = 0,
    .payload = NULL,
    .args = &sys_closure,
    .call = publish_stats
};

generate_uuid(sys_closure.closure_id);

/* Schedule as periodic task to be executed every N seconds */
evloop_add_periodic_task(event_loop, conf->stats_pub_interval,
                         0, &sys_closure);

{% endhighlight %}

The `publish_stats` callback is called periodically every N seconds where N is
defined on a configuration global pointer that we're going to implement soon.

But let's add the callback first:

**src/server.c**

{% highlight c %}

static void publish_message(unsigned short pkt_id,
                            unsigned short topiclen,
                            const char *topic,
                            unsigned short payloadlen,
                            unsigned char *payload) {

    /* Retrieve the Topic structure from the global map, exit if not found */
    struct topic *t = sol_topic_get(&sol, topic);

    if (!t)
        return;

    /* Build MQTT packet with command PUBLISH */
    union mqtt_packet pkt;
    struct mqtt_publish *p = mqtt_packet_publish(PUBLISH, pkt_id,
                                                 topiclen,
                                                 (unsigned char *) topic,
                                                 payloadlen,
                                                 payload);

    pkt.publish = *p;

    size_t len;
    unsigned char *packed;

    /* Send payload through TCP to all subscribed clients of the topic */
    struct list_node *cur = t->subscribers->head;
    size_t sent = 0L;
    for (; cur; cur = cur->next) {

        sol_debug("Sending PUBLISH (d%i, q%u, r%i, m%u, %s, ... (%i bytes))",
                  pkt.publish.header.bits.dup,
                  pkt.publish.header.bits.qos,
                  pkt.publish.header.bits.retain,
                  pkt.publish.pkt_id,
                  pkt.publish.topic,
                  pkt.publish.payloadlen);

        len = MQTT_HEADER_LEN + sizeof(uint16_t) +
            pkt.publish.topiclen + pkt.publish.payloadlen;

        struct subscriber *sub = cur->data;
        struct sol_client *sc = sub->client;

        /* Update QoS according to subscriber's one */
        pkt.publish.header.bits.qos = sub->qos;

        if (pkt.publish.header.bits.qos > AT_MOST_ONCE)
            len += sizeof(uint16_t);

        packed = pack_mqtt_packet(&pkt, PUBLISH_TYPE);

        if ((sent = send_bytes(sc->fd, packed, len)) < 0)
            sol_error("Error publishing to %s: %s",
                      sc->client_id, strerror(errno));

        // Update information stats
        info.bytes_sent += sent;
        info.messages_sent++;

        free(packed);
    }

    free(p);
}

/*
 * Publish statistics periodic task, it will be called once every N config
 * defined seconds, it publish some informations on predefined topics
 */
static void publish_stats(struct evloop *loop, void *args) {

    char cclients[number_len(info.nclients) + 1];
    sprintf(cclients, "%d", info.nclients);

    char bsent[number_len(info.bytes_sent) + 1];
    sprintf(bsent, "%lld", info.bytes_sent);

    char msent[number_len(info.messages_sent) + 1];
    sprintf(msent, "%lld", info.messages_sent);

    char mrecv[number_len(info.messages_recv) + 1];
    sprintf(mrecv, "%lld", info.messages_recv);

    long long uptime = time(NULL) - info.start_time;
    char utime[number_len(uptime) + 1];
    sprintf(utime, "%lld", uptime);

    double sol_uptime = (double)(time(NULL) - info.start_time) / SOL_SECONDS;
    char sutime[16];
    sprintf(sutime, "%.4f", sol_uptime);

    publish_message(0, strlen(sys_topics[5]), sys_topics[5],
                    strlen(utime), (unsigned char *) &utime);
    publish_message(0, strlen(sys_topics[6]), sys_topics[6],
                    strlen(sutime), (unsigned char *) &sutime);
    publish_message(0, strlen(sys_topics[7]), sys_topics[7],
                    strlen(cclients), (unsigned char *) &cclients);
    publish_message(0, strlen(sys_topics[9]), sys_topics[9],
                    strlen(bsent), (unsigned char *) &bsent);
    publish_message(0, strlen(sys_topics[11]), sys_topics[11],
                    strlen(msent), (unsigned char *) &msent);
    publish_message(0, strlen(sys_topics[12]), sys_topics[12],
                    strlen(mrecv), (unsigned char *) &mrecv);
}

{% endhighlight %}

Ok now we have our first periodic callback, it publishes general informations on
the status of the broker to a set of topics called `$SYS` topics, that we
called `$SOL` breaking the standards in a blink of an eye. These informations
could be added incrementally in the future.


**src/server.c**

{% highlight c %}

/*
 * General informations of the broker, all fields will be published
 * periodically to internal topics
 */
static struct sol_info info;

/* Broker global instance, contains the topic trie and the clients hashtable */
static struct sol sol;

{% endhighlight %}

There're still some parts that we have to write in order to have this piece of
code to compile and work, for example, what is that `closure_destructor`
function? What about that `info` structure that we update in `on_write` and
`on_read`? We can see that those have to do with some calls to `hashtable_*`
and `sol_topic_*`, which will be plugged-in soon.

Let's move forward to [part 4]({{site.url}}{{site.baseurl}}/2019/03/07/sol-mqtt-broker-p4),
we'll start implementing some handlers for every MQTT command.
