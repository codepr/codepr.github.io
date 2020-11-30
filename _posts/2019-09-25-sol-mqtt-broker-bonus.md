---
layout: post
title: "Sol - An MQTT broker from scratch. Refactoring & eventloop"
description: "Writing an MQTT broker from scratch, to really understand something you have to build it."
categories: c unix tutorial epoll
---

***UPDATE: 2020-02-07***

In the previous 6 parts we explored a fair amount of common CS topics such
as networks and data structures, the little journey ended up with a bugged but
working toy to play with.
<!--more-->
Out of curiosity, I wanted to test how close to something usable this pet
project could get and decided to make a huge refactoring to something less
hacky and a bit more structured, paying also attention to portability.
I won't walk through all the refactoring process, it would be deadly boring,
I'll just highlight some of the most important parts that needed adjustements,
the rest can be safely applied by merging the `master` branch into the
`tutorial` one or directly cloning it.

First I listed the main points that needed to be refined in order of priority:

- The low level I/O handling, correctly read/write stream of bytes
- Abstract over EPOLL as it's a Linux only interface, provide some kind of
  fallback
- Manage encryption, achieving a transparent interface for plain or
  encrypted communication
- Client session handling and peculiar features of a MQTT broker like '+'
  wildcard subscriptions

*Note:* *even tho the Hashtable implemented from scratch earlier was working fine
      I decided to move to a battle-tested and more performant `UTHASH` library,
      being it single header it was quiet simple to integrate it.
      See [https://troydhanson.github.io/uthash/](https://troydhanson.github.io/uthash/)
      for more info.*

#### Packets fragmentation is not funny

The first and foremost aspect to check was the network communication, by mainly
testing in local I only noticed after some heavier benchmarking that sometimes
the system was losing some packets, or rather, the kernel buffer was probably
flooded and started to fragment some payloads, TCP is after all a stream
protocol and it's perfectly fine to segment data during sending. It was a
little naive by me to not handle this properly initially, mostly out of rush to
reach something that could work without worrying of low level details, anyway
this led to some nasty behaviours, like packets wrongly parsed, being the first
byte of incoming chunk of data, recognized as a different than expected
instruction.<br/>
So one of the most important fix was the `recv_packet` function on the **server.c**
module, specifically the addition of a state-machine like behaviour for each
client, correctly performing non-blocking reads and writes without blocking
the thread ever.

I also moved core parts of the application, specifically main MQTT abstractions
like client session and topics to an "internal" header.

<hr>
**src/sol_internal.h**
<hr>

{% highlight c %}
/*
 * The client actions can be summarized as a roughly simple state machine,
 * comprised by 4 states:
 * - WAITING_HEADER it's the base state, waiting for the next packet to be
 *                  received
 * - WAITING_LENGTH the second state, a packet has arrived but it's not
 *                  complete yet. Accorting to MQTT protocol, after the first
 *                  byte we need to wait 1 to 4 more bytes based on the
 *                  encoded length (use continuation bit to state the number
 *                  of bytes needed, see http://docs.oasis-open.org/mqtt/mqtt/
 *                  v3.1.1/os/mqtt-v3.1.1-os.html for more info)
 * - WAITING_DATA   it's the step required to receive the full byte stream as
 *                  the encoded length describe. We wait for the effective
 *                  payload in this state.
 * - SENDING_DATA   the last status, a complete packet has been received and
 *                  has to be processed and reply back if needed.
 */
enum client_status {
    WAITING_HEADER,
    WAITING_LENGTH,
    WAITING_DATA,
    SENDING_DATA
};

/*
 * Wrapper structure around a connected client, each client can be a publisher
 * or a subscriber, it can be used to track sessions too.
 * As of now, no allocations will be fired, jsut a big pool of memory at the
 * start of the application will serve us a client pool, read and write buffers
 * are initialized lazily.
 *
 * It's an hashable struct which will be tracked during the execution of the
 * application, see https://troydhanson.github.io/uthash/userguide.html.
 */
struct client {
    struct ev_ctx *ctx; /* An event context refrence mostly used to fire write events */
    int rc;  /* Return code of the message just handled */
    int status; /* Current status of the client (state machine) */
    int rpos; /* The nr of bytes to skip after a complete packet has been read.
               * This because according to MQTT, length is encoded on multiple
               * bytes according to it's size, using continuation bit as a
               * technique to encode it. We don't want to decode the length two
               * times when we already know it, so we need an offset to know
               * where the actual packet will start
               */
    size_t read; /* The number of bytes already read */
    size_t toread; /* The number of bytes that have to be read */
    unsigned char *rbuf; /* The reading buffer */
    size_t wrote; /* The number of bytes already written */
    size_t towrite; /* The number of bytes we have to write */
    unsigned char *wbuf; /* The writing buffer */
    char client_id[MQTT_CLIENT_ID_LEN]; /* The client ID according to MQTT specs */
    struct connection conn; /* A connection structure, takes care of plain or
                             * TLS encrypted communication by using callbacks
                             */
    struct client_session *session; /* The session associated to the client */
    unsigned long last_seen; /* The timestamp of the last action performed */
    bool online;  /* Just an online flag */
    bool connected; /* States if the client has already processed a connection packet */
    bool has_lwt; /* States if the connection packet carried a LWT message */
    bool clean_session; /* States if the connection packet was set to clean session */
    UT_hash_handle hh; /* UTHASH handle, needed to use UTHASH macros */
};

/*
 * Every client has a session which track his subscriptions, possible missed
 * messages during disconnection time (that iff clean_session is set to false),
 * inflight messages and the message ID for each one.
 * A maximum of 65535 mid can be used at the same time according to MQTT specs,
 * so i_acks, i_msgs and in_i_acks, thus being allocated on the heap during the
 * init, will be of 65535 length each.
 *
 * It's a hashable struct that will be tracked during the entire lifetime of
 * the application, governed by the clean_session flag on connection from
 * clients
 */
struct client_session {
    int next_free_mid; /* The next 'free' message ID */
    List *subscriptions; /* All the clients subscriptions, stored as topic structs */
    List *outgoing_msgs; /* Outgoing messages during disconnection time, stored as mqtt_packet pointers */
    bool has_inflight; /* Just a flag stating the presence of inflight messages */
    bool clean_session; /* Clean session flag */
    char session_id[MQTT_CLIENT_ID_LEN]; /* The client_id the session refers to */
    struct mqtt_packet lwt_msg; /* A possibly NULL LWT message, will be set on connection */
    struct inflight_msg *i_acks; /* Inflight ACKs that must be cleared */
    struct inflight_msg *i_msgs; /* Inflight MSGs that must be sent out DUP in case of timeout */
    struct inflight_msg *in_i_acks; /* Inflight input ACKs that must be cleared by the client */
    UT_hash_handle hh; /* UTHASH handle, needed to use UTHASH macros */
    struct ref refcount; /* Reference counting struct, to share the struct easily */
};

{% endhighlight %}
<hr>

So the client structure is a bit more beefy now and it stores the status of
each packet read/write in order to resume it in case of `EAGAIN` errors from
the kernel space.

<hr>
**src/server.c**
<hr>

{% highlight c %}
static ssize_t recv_packet(struct client *c) {
    ssize_t nread = 0;
    unsigned opcode = 0, pos = 0;
    unsigned long long pktlen = 0LL;
    // Base status, we have read 0 to 2 bytes
    if (c->status == WAITING_HEADER) {
        /*
         * Read the first two bytes, the first should contain the message type
         * code
         */
        nread = recv_data(&c->conn, c->rbuf + c->read, 2 - c->read);
        if (errno != EAGAIN && errno != EWOULDBLOCK && nread <= 0)
            return -ERRCLIENTDC;
        c->read += nread;
        if (errno == EAGAIN && c->read < 2)
            return -ERREAGAIN;
        c->status = WAITING_LENGTH;
    }
    /*
     * We have already read the packet HEADER, thus we know what packet we're
     * dealing with, we're between bytes 2-4, as after the 1st byte, the
     * remaining 3 can be all used to store the packet length, or, in case of
     * ACK type packet or PINGREQ/PINGRESP and DISCONNECT, the entire packet
     */
    if (c->status == WAITING_LENGTH) {
        if (c->read == 2) {
            opcode = *c->rbuf >> 4;
            /*
             * Check for OPCODE, if an unknown OPCODE is received return an
             * error
             */
            if (DISCONNECT < opcode || CONNECT > opcode)
                return -ERRPACKETERR;
            /*
             * We have a PINGRESP/PINGREQ or a DISCONNECT packet, we're done
             * here
             */
            if (opcode > UNSUBSCRIBE) {
                c->rpos = 2;
                c->toread = c->read;
                goto exit;
            }
        }
        /*
         * Read 2 extra bytes, because the first 4 bytes could countain the
         * total size in bytes of the entire packet
         */
        nread = recv_data(&c->conn, c->rbuf + c->read, 4 - c->read);
        if (errno != EAGAIN && errno != EWOULDBLOCK && nread <= 0)
            return -ERRCLIENTDC;
        c->read += nread;
        if (errno == EAGAIN && c->read < 4)
            return -ERREAGAIN;
        /*
         * Read remaining length bytes which starts at byte 2 and can be long to
         * 4 bytes based on the size stored, so byte 2-5 is dedicated to the
         * packet length.
         */
        pktlen = mqtt_decode_length(c->rbuf + 1, &pos);
        /*
         * Set return code to -ERRMAXREQSIZE in case the total packet len
         * exceeds the configuration limit `max_request_size`
         */
        if (pktlen > conf->max_request_size)
            return -ERRMAXREQSIZE;
        /*
         * Update the toread field for the client with the entire length of
         * the current packet, which is comprehensive of packet length,
         * bytes used to encode it and 1 byte for the header
         * We've already tracked the bytes we read so far, we just need to
         * read toread-read bytes.
         */
        c->rpos = pos + 1;
        c->toread = pktlen + pos + 1;  // pos = bytes used to store length
        /* Looks like we got an ACK packet, we're done reading */
        if (pktlen <= 4)
            goto exit;
        c->status = WAITING_DATA;
    }
    /*
     * Last status, we have access to the length of the packet and we know for
     * sure that it's not a PINGREQ/PINGRESP/DISCONNECT packet.
     */
    nread = recv_data(&c->conn, c->rbuf + c->read, c->toread - c->read);
    if (errno != EAGAIN && errno != EWOULDBLOCK && nread <= 0)
        return -ERRCLIENTDC;
    c->read += nread;
    if (errno == EAGAIN && c->read < c->toread)
        return -ERREAGAIN;
exit:
    return 0;
}
/*
 * Handle incoming requests, after being accepted or after a reply, under the
 * hood it calls recv_packet and return an error code according to the outcome
 * of the operation
 */
static inline int read_data(struct client *c) {
    /*
     * We must read all incoming bytes till an entire packet is received. This
     * is achieved by following the MQTT protocol specifications, which
     * send the size of the remaining packet as the second byte. By knowing it
     * we know if the packet is ready to be deserialized and used.
     */
    int err = recv_packet(c);
    /*
     * Looks like we got a client disconnection or If a not correct packet
     * received, we must free the buffer and reset the handler to the request
     * again, setting EPOLL to EPOLLIN
     *
     * TODO: Set a error_handler for ERRMAXREQSIZE instead of dropping client
     *       connection, explicitly returning an informative error code to the
     *       client connected.
     */
    if (err < 0)
        goto err;
    if (c->read < c->toread)
        return -ERREAGAIN;
    info.bytes_recv += c->read;
    return 0;
    // Disconnect packet received
err:
    return err;
}
/*
 * Write stream of bytes to a client represented by a connection object, till
 * all bytes to be written is exhausted, tracked by towrite field or if an
 * EAGAIN (socket descriptor must be in non-blocking mode) error is raised,
 * meaning we cannot write anymore for the current cycle.
 */
static inline int write_data(struct client *c) {
    ssize_t wrote = send_data(&c->conn, c->wbuf+c->wrote, c->towrite-c->wrote);
    if (errno != EAGAIN && errno != EWOULDBLOCK && wrote < 0)
        return -ERRCLIENTDC;
    c->wrote += wrote > 0 ? wrote : 0;
    if (c->wrote < c->towrite && errno == EAGAIN)
        return -ERREAGAIN;
    // Update information stats
    info.bytes_sent += c->towrite;
    // Reset client written bytes track fields
    c->towrite = c->wrote = 0;
    return 0;
}
{% endhighlight %}
<hr>

Worth a note, `recv_packet` and `write_data` functions calls in turn two lower
level functions defined in the **network.h** header:

- `ssize_t send_data(struct connection *, const unsigned char *, size_t)`
- `ssize_t recv_data(struct connection *, unsigned char *, size_t)`

Both functions accept a `struct connection` as first parameter, the other two
are simply the buffer to be filled/emptied and the number of bytes to
read/write.

That connection structure directly address the 3rd point described earlier in
the improvements list, it's an abstraction over a socket connection with a
client and it is comprised of 4 fundamental callbacks neeed to manage the
communication:

- accept
- send
- recv
- close

This approach allows to build a new connection for each connecting client based
on the communication type chosen, be it plain or TLS encrypted, allowing to use
just a single function in both cases.
It's definition is:

<hr>
**src/network.h**
<hr>

{% highlight c %}
/*
 * Connection abstraction struct, provide a transparent interface for
 * connection handling, taking care of communication layer, being it encrypted
 * or plain, by setting the right callbacks to be used.
 *
 * The 4 main operations reflected by those callbacks are the ones that can be
 * performed on every FD:
 *
 * - accept
 * - read
 * - write
 * - close
 *
 * According to the type of connection we need, each one of these actions will
 * be set with the right function needed. Maintain even the address:port of the
 * connecting client.
 */
struct connection {
    int fd;
    SSL *ssl;
    SSL_CTX *ctx;
    char ip[INET_ADDRSTRLEN + 6];
    int (*accept) (struct connection *, int);
    ssize_t (*send) (struct connection *, const unsigned char *, size_t);
    ssize_t (*recv) (struct connection *, unsigned char *, size_t);
    void (*close) (struct connection *);
};
{% endhighlight %}
<hr>

It also stores an `SSL *` and an `SSL_CTX *`, those are left `NULL` in case of
plain communication.

Another good improvement was the correction of the packing and unpacking
functions (thanks to [beej networking
guide](https://beej.us/guide/bgnet/html/single/bgnet.html#serialization), this
guide is pure gold) and the addition of some helper functions to handle
integer and bytes unpacking:

<hr>
**src/pack.c**
<hr>

{% highlight c %}
/* Helper functions */
long long unpack_integer(unsigned char **buf, char size) {
    long long val = 0LL;
    switch (size) {
        case 'b':
            val = **buf;
            *buf += 1;
            break;
        case 'B':
            val = **buf;
            *buf += 1;
            break;
        case 'h':
            val = unpacki16(*buf);
            *buf += 2;
            break;
        case 'H':
            val = unpacku16(*buf);
            *buf += 2;
            break;
        case 'i':
            val = unpacki32(*buf);
            *buf += 4;
            break;
        case 'I':
            val = unpacku32(*buf);
            *buf += 4;
            break;
        case 'q':
            val = unpacki64(*buf);
            *buf += 8;
            break;
        case 'Q':
            val = unpacku64(*buf);
            *buf += 8;
            break;
    }
    return val;
}

unsigned char *unpack_bytes(unsigned char **buf, size_t len) {
    unsigned char *dest = malloc(len + 1);
    memcpy(dest, *buf, len);
    dest[len] = '\0';
    *buf += len;
    return dest;
}
{% endhighlight %}
<hr>

### Adding a tiny eventloop: ev

Abstracting over the multiplexing APIs offered by the host machine has not been
that hard in a single threaded context, it essentially consists of a structure
tracking a range of events by using a pre-allocated array of custom
struct events. The header is pretty descriptive, the most important parts to
grasp are the standardization of events into our convention (see `enum ev_type`)
the custom event struct (`struct ev`) and the `events_monitored` array that
pre-allocate a number of events corresponding to multiplexing events fired
from the kernel space (e.g. if a descriptor is ready to read some data or to
write).

Using an opaque `void *` pointer allows us to plug-in whatever underlying API
the host machine provide, be it `EPOLL`, `SELECT` or `KQUEUE`.

<hr>
**src/ev.h**
<hr>
{% highlight c %}
#include <sys/time.h>
#define EV_OK  0
#define EV_ERR 1
/*
 * Event types, meant to be OR-ed on a bitmask to define the type of an event
 * which can have multiple traits
 */
enum ev_type {
    EV_NONE       = 0x00,
    EV_READ       = 0x01,
    EV_WRITE      = 0x02,
    EV_DISCONNECT = 0x04,
    EV_EVENTFD    = 0x08,
    EV_TIMERFD    = 0x10,
    EV_CLOSEFD    = 0x20
};
struct ev_ctx;
/*
 * Event struture used as the main carrier of clients informations, it will be
 * tracked by an array in every context created
 */
struct ev {
    int fd;
    int mask;
    void *rdata; // opaque pointer for read callback args
    void *wdata; // opaque pointer for write callback args
    void (*rcallback)(struct ev_ctx *, void *); // read callback
    void (*wcallback)(struct ev_ctx *, void *); // write callback
};
/*
 * Event loop context, carry the expected number of events to be monitored at
 * every cycle and an opaque pointer to the backend used as engine
 * (Select | Epoll | Kqueue).
 * By now we stick with epoll and skip over select, cause as the current
 * threaded model employed by the server is not very friendly with select
 * Level-trigger default setting. But it would be quiet easy abstract over the
 * select model as well for single threaded uses or in a loop per thread
 * scenario (currently thanks to epoll Edge-triggered + EPOLLONESHOT we can
 * share a single loop over multiple threads).
 */
struct ev_ctx {
    int events_nr;
    int maxfd; // the maximum FD monitored by the event context,
               // events_monitored must be at least maxfd long
    int stop;
    int maxevents;
    unsigned long long fired_events;
    struct ev *events_monitored;
    void *api; // opaque pointer to platform defined backends
};
void ev_init(struct ev_ctx *, int);
void ev_destroy(struct ev_ctx *);
/*
 * Poll an event context for events, accepts a timeout or block forever,
 * returning only when a list of FDs are ready to either READ, WRITE or TIMER
 * to be executed.
 */
int ev_poll(struct ev_ctx *, time_t);
/*
 * Blocks forever in a loop polling for events with ev_poll calls. At every
 * cycle executes callbacks registered with each event
 */
int ev_run(struct ev_ctx *);
/*
 * Trigger a stop on a running event, it's meant to be run as an event in a
 * running ev_ctx
 */
void ev_stop(struct ev_ctx *);
/*
 * Add a single FD to the underlying backend of the event loop. Equal to
 * ev_fire_event just without an event to be carried. Useful to add simple
 * descritors like a listening socket o message queue FD.
 */
int ev_watch_fd(struct ev_ctx *, int, int);
/*
 * Remove a FD from the loop, even tho a close syscall is sufficient to remove
 * the FD from the underlying backend such as EPOLL/SELECT, this call ensure
 * that any associated events is cleaned out an set to EV_NONE
 */
int ev_del_fd(struct ev_ctx *, int);
/*
 * Register a new event, semantically it's equal to ev_register_event but
 * it's meant to be used when an FD is not already watched by the event loop.
 * It could be easily integrated in ev_fire_event call but I prefer maintain
 * the samantic separation of responsibilities.
 */
int ev_register_event(struct ev_ctx *, int, int,
                      void (*callback)(struct ev_ctx *, void *), void *);
int ev_register_cron(struct ev_ctx *,
                     void (*callback)(struct ev_ctx *, void *),
                     void *,
                     long long, long long);
/*
 * Register a new event for the next loop cycle to a FD. Equal to ev_watch_fd
 * but allow to carry an event object for the next cycle.
 */
int ev_fire_event(struct ev_ctx *, int, int,
                  void (*callback)(struct ev_ctx *, void *), void *);
{% endhighlight %}
<hr>

At the init of the server, the `ev_ctx` will be instructed to run some
periodic tasks and to run a callback on accept on new connections. From now
on start a simple juggling of callbacks to be scheduled on the event loop,
typically after being accepted a connection his handle (fd) will be added to
the backend of the loop (this case we're using `EPOLL` as a backend but also
`KQUEUE` or `SELECT/POLL` should be easy to plug-in) and read_callback will be
run every time there's new data incoming. If a complete packet is received
and correctly parsed it will be processed by calling the right handler from
the handler module, based on the command it carries and a response will be
fired back.

{% highlight bash %}

                             MAIN THREAD
                              [EV_CTX]

    ACCEPT_CALLBACK         READ_CALLBACK         WRITE_CALLBACK
  -------------------    ------------------    --------------------
           |                     |                       |
        ACCEPT                   |                       |
           | ------------------> |                       |
           |               READ AND DECODE               |
           |                     |                       |
           |                     |                       |
           |                  PROCESS                    |
           |                     |                       |
           |                     |                       |
           |                     | --------------------> |
           |                     |                     WRITE
        ACCEPT                   |                       |
           | ------------------> | <-------------------- |
           |                     |                       |
{% endhighlight %}

This is the lifecycle of a connecting client, we got an accept-only callback
that demand IO handling to read and write callbacks till the disconnection of
the client.

The mentioned callbacks have been added to the server module and they're
extremely simple, a thing always appreciated

<hr>
**src/server.c**
<hr>

{% highlight c %}
/*
 * Handle incoming connections, create a a fresh new struct client structure
 * and link it to the fd, ready to be set in EV_READ event, then schedule a
 * call to the read_callback to handle incoming streams of bytes
 */
static void accept_callback(struct ev_ctx *ctx, void *data) {
    int serverfd = *((int *) data);
    while (1) {
        /*
         * Accept a new incoming connection assigning ip address
         * and socket descriptor to the connection structure
         * pointer passed as argument
         */
        struct connection conn;
        connection_init(&conn, conf->tls ? server.ssl_ctx : NULL);
        int fd = accept_connection(&conn, serverfd);
        if (fd == 0)
            continue;
        if (fd < 0) {
            close_connection(&conn);
            break;
        }
        /*
         * Create a client structure to handle his context
         * connection
         */
        struct client *c = memorypool_alloc(server.pool);
        c->conn = conn;
        client_init(c);
        c->ctx = ctx;
        /* Add it to the epoll loop */
        ev_register_event(ctx, fd, EV_READ, read_callback, c);
        /* Record the new client connected */
        info.nclients++;
        info.nconnections++;
        log_info("[%p] Connection from %s", (void *) pthread_self(), conn.ip);
    }
}
/*
 * Reading packet callback, it's the main function that will be called every
 * time a connected client has some data to be read, notified by the eventloop
 * context.
 */
static void read_callback(struct ev_ctx *ctx, void *data) {
    struct client *c = data;
    if (c->status == SENDING_DATA)
        return;
    /*
     * Received a bunch of data from a client, after the creation
     * of an IO event we need to read the bytes and encoding the
     * content according to the protocol
     */
    int rc = read_data(c);
    switch (rc) {
        case 0:
            /*
             * All is ok, raise an event to the worker poll EPOLL and
             * link it with the IO event containing the decode payload
             * ready to be processed
             */
            /* Record last action as of now */
            c->last_seen = time(NULL);
            c->status = SENDING_DATA;
            process_message(ctx, c);
            break;
        case -ERRCLIENTDC:
        case -ERRPACKETERR:
        case -ERRMAXREQSIZE:
            /*
             * We got an unexpected error or a disconnection from the
             * client side, remove client from the global map and
             * free resources allocated such as io_event structure and
             * paired payload
             */
            log_error("Closing connection with %s (%s): %s",
                      c->client_id, c->conn.ip, solerr(rc));
            // Publish, if present, LWT message
            if (c->has_lwt == true) {
                char *tname = (char *) c->session->lwt_msg.publish.topic;
                struct topic *t = topic_get(&server, tname);
                publish_message(&c->session->lwt_msg, t);
            }
            // Clean resources
            ev_del_fd(ctx, c->conn.fd);
            // Remove from subscriptions for now
            if (c->session && list_size(c->session->subscriptions) > 0) {
                struct list *subs = c->session->subscriptions;
                list_foreach(item, subs) {
                    log_debug("Deleting %s from topic %s",
                              c->client_id, ((struct topic *) item->data)->name);
                    topic_del_subscriber(item->data, c);
                }
            }
            client_deactivate(c);
            info.nclients--;
            info.nconnections--;
            break;
        case -ERREAGAIN:
            ev_fire_event(ctx, c->conn.fd, EV_READ, read_callback, c);
            break;
    }
}
/*
 * This function is called only if the client has sent a full stream of bytes
 * consisting of a complete packet as expected by the MQTT protocol and by the
 * declared length of the packet.
 * It uses eventloop APIs to react accordingly to the packet type received,
 * validating it before proceed to call handlers. Depending on the handler
 * called and its outcome, it'll enqueue an event to write a reply or just
 * reset the client state to allow reading some more packets.
 */
static void process_message(struct ev_ctx *ctx, struct client *c) {
    struct io_event io = { .client = c };
    /*
     * Unpack received bytes into a mqtt_packet structure and execute the
     * correct handler based on the type of the operation.
     */
    mqtt_unpack(c->rbuf + c->rpos, &io.data, *c->rbuf, c->read - c->rpos);
    c->toread = c->read = c->rpos = 0;
    c->rc = handle_command(io.data.header.bits.type, &io);
    switch (c->rc) {
        case REPLY:
        case MQTT_NOT_AUTHORIZED:
        case MQTT_BAD_USERNAME_OR_PASSWORD:
            /*
             * Write out to client, after a request has been processed in
             * worker thread routine. Just send out all bytes stored in the
             * reply buffer to the reply file descriptor.
             */
            enqueue_event_write(c);
            /* Free resource, ACKs will be free'd closing the server */
            if (io.data.header.bits.type != PUBLISH)
                mqtt_packet_destroy(&io.data);
            break;
        case -ERRCLIENTDC:
            ev_del_fd(ctx, c->conn.fd);
            client_deactivate(io.client);
            // Update stats
            info.nclients--;
            info.nconnections--;
            break;
        case -ERRNOMEM:
            log_error(solerr(c->rc));
            break;
        default:
            c->status = WAITING_HEADER;
            if (io.data.header.bits.type != PUBLISH)
                mqtt_packet_destroy(&io.data);
            break;
    }
}
/*
 * Callback dedicated to client replies, try to send as much data as possible
 * epmtying the client buffer and rearming the socket descriptor for reading
 * after
 */
static void write_callback(struct ev_ctx *ctx, void *arg) {
    struct client *client = arg;
    int err = write_data(client);
    switch (err) {
        case 0: // OK
            /*
             * Rearm descriptor making it ready to receive input,
             * read_callback will be the callback to be used; also reset the
             * read buffer status for the client.
             */
            client->status = WAITING_HEADER;
            ev_fire_event(ctx, client->conn.fd, EV_READ, read_callback, client);
            break;
        case -ERREAGAIN:
            enqueue_event_write(client);
            break;
        default:
            log_info("Closing connection with %s (%s): %s %i",
                     client->client_id, client->conn.ip,
                     solerr(client->rc), err);
            ev_del_fd(ctx, client->conn.fd);
            client_deactivate(client);
            // Update stats
            info.nclients--;
            info.nconnections--;
            break;
    }
}
{% endhighlight %}
<hr>

Of course the starting server will have to make a blocking call starting the
eventloop, and we'll need a stop mechanism as well, thanks to `ev_stop` API it
has been pretty simple to add an additional event routine to be called when we
want to stop the running loop.

<hr>
**src/server.c**
<hr>

{% highlight c %}
/*
 * Eventloop stop callback, will be triggered by an EV_CLOSEFD event and stop
 * the running loop, unblocking the call.
 */
static void stop_handler(struct ev_ctx *ctx, void *arg) {
    (void) arg;
    ev_stop(ctx);
}
/*
 * IO worker function, wait for events on a dedicated epoll descriptor which
 * is shared among multiple threads for input and output only, following the
 * normal EPOLL semantic, EPOLLIN for incoming bytes to be unpacked and
 * processed by a worker thread, EPOLLOUT for bytes incoming from a worker
 * thread, ready to be delivered out.
 */
static void eventloop_start(void *args) {
    int sfd = *((int *) args);
    struct ev_ctx ctx;
    ev_init(&ctx, EVENTLOOP_MAX_EVENTS);
    // Register stop event
    ev_register_event(&ctx, conf->run, EV_CLOSEFD|EV_READ, stop_handler, NULL);
    // Register listening FD with accept callback
    ev_register_event(&ctx, sfd, EV_READ, accept_callback, &sfd);
    // Register periodic tasks
    ev_register_cron(&ctx, publish_stats, NULL, conf->stats_pub_interval, 0);
    ev_register_cron(&ctx, inflight_msg_check, NULL, 0, 9e8);
    // Start the loop, blocking call
    ev_run(&ctx);
    ev_destroy(&ctx);
}
/* Fire a write callback to reply after a client request */
void enqueue_event_write(const struct client *c) {
    ev_fire_event(c->ctx, c->conn.fd, EV_WRITE, write_callback, (void *) c);
}
{% endhighlight %}
<hr>

So the final `start_server` function, which is one of the two exposed APIs of
the server module will just be changed to start an eventloop with an opened
socket in listening mode:

<hr>
**src/server.c**
<hr>

{% highlight c %}
/*
 * Main entry point for the server, to be called with an address and a port
 * to start listening
 */
int start_server(const char *addr, const char *port) {
    /* Initialize global Sol instance */
    trie_init(&server.topics, NULL);
    server.authentications = NULL;
    server.pool = memorypool_new(BASE_CLIENTS_NUM, sizeof(struct client));
    server.clients_map = NULL;
    server.sessions = NULL;
    server.wildcards = list_new(wildcard_destructor);
    if (conf->allow_anonymous == false)
        config_read_passwd_file(conf->password_file, &server.authentications);
    /* Generate stats topics */
    for (int i = 0; i < SYS_TOPICS; i++)
        topic_put(&server, topic_new(xstrdup(sys_topics[i].name)));
    /* Start listening for new connections */
    int sfd = make_listen(addr, port, conf->socket_family);
    /* Setup SSL in case of flag true */
    if (conf->tls == true) {
        openssl_init();
        server.ssl_ctx = create_ssl_context();
        load_certificates(server.ssl_ctx, conf->cafile,
                          conf->certfile, conf->keyfile);
    }
    log_info("Server start");
    info.start_time = time(NULL);
    // start eventloop, could be spread on multiple threads
    eventloop_start(&sfd);
    close(sfd);
    AUTH_DESTROY(server.authentications);
    list_destroy(server.wildcards, 1);
    /* Destroy SSL context, if any present */
    if (conf->tls == true) {
        SSL_CTX_free(server.ssl_ctx);
        openssl_cleanup();
    }
    log_info("Sol v%s exiting", VERSION);
    return 0;
}
{% endhighlight %}
<hr>

Commands are handled through a dispatch table, a common pattern used in C where
we map function pointers inside an array, in this case each position in the
array corresponds to an MQTT command.<br>
As you can see, there's also a `memorypool_new` call for clients, I decided
to pre-allocate a fixed number of clients, allowing the reuse of them when
disconnection occurs, the memory cost is negligible and totally worth it, as
long as the connecting clients are lazily inited, specifically their read and
write buffer, which can also be MB size.

This of course is only a fraction of what the ordeal has been but eventually I
came up with a decent prototype, the next step will be to stress test it a bit
and see how it goes compared to the battle-tested and indisputably better
pieces of software like Mosquitto or Mosca. Lot of missing features still, like
a persistence layer for session storing, but the mere pub/sub part should be
testable. Hopefully, this tutorial would work as a starting point for something
neater and carefully designed. Cya.
