---
layout: post
title: "Sol - An MQTT broker from scratch. Part-2"
description: "Writing an MQTT broker from scratch, to really understand something you have to build it."
tags: [c, unix, tutorial]
---

Let's continue from where we left, our `src/mqtt.c` module has now all
unpacking functions, we must add the remaining build helpers and the packing
functions to serialize packet for output.

## Build, pack and send.

For now we only need `CONNACK`, `SUBACK` and `PUBLISH` packet builder, the
other `ACK` like packets can be created at the same manner with a single
function, that's why the use of `typedef` for different ack codes.

- `union mqtt_header *mqtt_packet_header(unsigned char)` will cover packet
  Fixed Header as well as `PINGREQ`, `PINGRESP` and `DISCONNECT` packets
- `struct mqtt_ack *mqtt_packet_ack(unsigned char, unsigned short)` will be
  used to  build:
  - `PUBACK`
  - `PUBREC`
  - `PUBREL`
  - `PUBCOMP`
  - `UNSUBACK`

The remaining packets will have a dedicated function. There's probably better
ways to reuse code and to model this but for now let's stick to something
working, time to optimize and refactor will come.

**src/mqtt.c**

{% highlight c %}

/*
 * MQTT packets building functions
 */

union mqtt_header *mqtt_packet_header(unsigned char byte) {

    static union mqtt_header header;

    header.byte = byte;

    return &header;
}


struct mqtt_ack *mqtt_packet_ack(unsigned char byte, unsigned short pkt_id) {

    static struct mqtt_ack ack;

    ack.header.byte = byte;
    ack.pkt_id = pkt_id;

    return &ack;
}


struct mqtt_connack *mqtt_packet_connack(unsigned char byte,
                                         unsigned char cflags,
                                         unsigned char rc) {

    static struct mqtt_connack connack;

	connack.header.byte = byte;
    connack.byte = cflags;
    connack.rc = rc;

	return &connack;
}


struct mqtt_suback *mqtt_packet_suback(unsigned char byte,
                                       unsigned short pkt_id,
                                       unsigned char *rcs,
                                       unsigned short rcslen) {

    struct mqtt_suback *suback = malloc(sizeof(*suback));

    suback->header.byte = byte;
    suback->pkt_id = pkt_id;
    suback->rcslen = rcslen;
    suback->rcs = (unsigned char *) strdup((const char *) rcs);

    return suback;
}



struct mqtt_publish *mqtt_packet_publish(unsigned char byte,
                                         unsigned short pkt_id,
                                         size_t topiclen,
                                         unsigned char *topic,
                                         size_t payloadlen,
                                         unsigned char *payload) {

    struct mqtt_publish *publish = malloc(sizeof(*publish));

    publish->header.byte = byte;
    publish->pkt_id = pkt_id;
    publish->topiclen = topiclen;
    publish->topic = topic;
    publish->payloadlen = payloadlen;
    publish->payload = payload;

    return publish;
}


void mqtt_packet_release(union mqtt_packet *pkt, unsigned type) {

    switch (type) {
        case CONNECT_TYPE:
            free(pkt->connect.payload.client_id);
            if (pkt->connect.bits.username == 1)
                free(pkt->connect.payload.username);
            if (pkt->connect.bits.password == 1)
                free(pkt->connect.payload.password);
            if (pkt->connect.bits.will == 1) {
                free(pkt->connect.payload.will_message);
                free(pkt->connect.payload.will_topic);
            }
            break;
        case SUBSCRIBE_TYPE:
        case UNSUBSCRIBE_TYPE:
            for (unsigned i = 0; i < pkt->subscribe.tuples_len; i++)
                free(pkt->subscribe.tuples[i].topic);
            free(pkt->subscribe.tuples);
            break;
        case SUBACK_TYPE:
            free(pkt->suback.rcs);
            break;
        case PUBLISH_TYPE:
            free(pkt->publish.topic);
            free(pkt->publish.payload);
            break;
        default:
            break;
    }
}

{% endhighlight %}

We move on to packing functions now, essentially they reflect unpacking one,
but on the other way around: We start from structs and union to build a
bytearray.

{% highlight c %}


/*
 * MQTT packets packing functions
 */

static unsigned char *pack_mqtt_header(const union mqtt_header *hdr) {

    unsigned char *packed = malloc(MQTT_HEADER_LEN);
    unsigned char *ptr = packed;

    pack_u8(&ptr, hdr->byte);

    /* Encode 0 length bytes, message like this have only a fixed header */
    mqtt_encode_length(ptr, 0);

    return packed;
}


static unsigned char *pack_mqtt_ack(const union mqtt_packet *pkt) {

    unsigned char *packed = malloc(MQTT_ACK_LEN);
    unsigned char *ptr = packed;

    pack_u8(&ptr, pkt->ack.header.byte);
    mqtt_encode_length(ptr, MQTT_HEADER_LEN);
    ptr++;

    pack_u16(&ptr, pkt->ack.pkt_id);

    return packed;
}


static unsigned char *pack_mqtt_connack(const union mqtt_packet *pkt) {

    unsigned char *packed = malloc(MQTT_ACK_LEN);
    unsigned char *ptr = packed;

    pack_u8(&ptr, pkt->connack.header.byte);
    mqtt_encode_length(ptr, MQTT_HEADER_LEN);
    ptr++;

    pack_u8(&ptr, pkt->connack.byte);
    pack_u8(&ptr, pkt->connack.rc);

    return packed;
}


static unsigned char *pack_mqtt_suback(const union mqtt_packet *pkt) {

    size_t pktlen = MQTT_HEADER_LEN + sizeof(uint16_t) + pkt->suback.rcslen;
    unsigned char *packed = malloc(pktlen + 0);
    unsigned char *ptr = packed;

    pack_u8(&ptr, pkt->suback.header.byte);
    size_t len = sizeof(uint16_t) + pkt->suback.rcslen;
    int step = mqtt_encode_length(ptr, len);
    ptr += step;

    pack_u16(&ptr, pkt->suback.pkt_id);
    for (int i = 0; i < pkt->suback.rcslen; i++)
        pack_u8(&ptr, pkt->suback.rcs[i]);

    return packed;
}


static unsigned char *pack_mqtt_publish(const union mqtt_packet *pkt) {

    /*
     * We must calculate the total length of the packet including header and
     * length field of the fixed header part
     */
    size_t pktlen = MQTT_HEADER_LEN + sizeof(uint16_t) +
        pkt->publish.topiclen + pkt->publish.payloadlen;

    // Total len of the packet excluding fixed header len
    size_t len = 0L;

    if (pkt->header.bits.qos > AT_MOST_ONCE)
        pktlen += sizeof(uint16_t);

    unsigned char *packed = malloc(pktlen);
    unsigned char *ptr = packed;

    pack_u8(&ptr, pkt->publish.header.byte);

    // Total len of the packet excluding fixed header len
    len += (pktlen - MQTT_HEADER_LEN);

    /*
     * TODO handle case where step is > 1, e.g. when a message longer than 128
     * bytes is published
     */
    int step = mqtt_encode_length(ptr, len);
    ptr += step;

    // Topic len followed by topic name in bytes
    pack_u16(&ptr, pkt->publish.topiclen);
    pack_bytes(&ptr, pkt->publish.topic);

    // Packet id
    if (pkt->header.bits.qos > AT_MOST_ONCE)
        pack_u16(&ptr, pkt->publish.pkt_id);

    // Finally the payload, same way of topic, payload len -> payload
    pack_bytes(&ptr, pkt->publish.payload);

    return packed;
}


unsigned char *pack_mqtt_packet(const union mqtt_packet *pkt, unsigned type) {

    if (type == PINGREQ_TYPE || type == PINGRESP_TYPE)
        return pack_mqtt_header(&pkt->header);

    return pack_handlers[type](pkt);
}

{% endhighlight %}


## The server

The server we're gonna create will be a single-threaded TCP server with
multiplexed I/O by using `EPOLL` interface. `EPOLL` is the last multiplexing
mechanism after `SELECT` and `POLL` added with kernel 2.5.44, and the most
performant with high number of connection, it's counterpart for BSD and
BSD-like systems is `kqueue`.

We're gonna need some functions to manage our socket descritor.

**src/network.h**

{% highlight c %}

#ifndef NETWORK_H
#define NETWORK_H

#include <stdio.h>
#include <stdint.h>
#include "util.h"


// Socket families
#define UNIX    0
#define INET    1

/* Set non-blocking socket */
int set_nonblocking(int);

/*
 * Set TCP_NODELAY flag to true, disabling Nagle's algorithm, no more waiting
 * for incoming packets on the buffer
 */
int set_tcp_nodelay(int);

/* Auxiliary function for creating epoll server */
int create_and_bind(const char *, const char *, int);

/*
 * Create a non-blocking socket and make it listen on the specfied address and
 * port
 */
int make_listen(const char *, const char *, int);

/* Accept a connection and add it to the right epollfd */
int accept_connection(int);

{% endhighlight %}

Just some well-known helper functions to create and bind socket to listen for
new connections and to set socket in non-blocking mode (a requirement to use
`EPOLL` multiplexing at his best).

I don't like to have to manage all streams of bytes incoming to and exiting
from the host, this two functions never fail to appear in every C codebase
regarding TCP communication:

- `ssize_t send_bytes(int, const unsigned char *, size_t)` used to send all
  bytes out at once in while loop till no bytes left, by handling `EAGAIN` and
  `EWOUDLBLOCK` error codes
- `ssize_t recv_bytes(int, unsigned char *, size_t)`, read an arbitrary number
  of bytes in a while loop, again handling correctly `EAGAIN` and `EWOUDLBLOCK`
  error codes


**src/network.h**

{% highlight c %}

/* I/O management functions */

/*
 * Send all data in a loop, avoiding interruption based on the kernel buffer
 * availability
 */
ssize_t send_bytes(int, const unsigned char *, size_t);

/*
 * Receive (read) an arbitrary number of bytes from a file descriptor and
 * store them in a buffer
 */
ssize_t recv_bytes(int, unsigned char *, size_t);


#endif

{% endhighlight %}

And the implementation on networ.c.

**src/network.c**

{% highlight c %}

#define _DEFAULT_SOURCE
#include <stdlib.h>
#include <errno.h>
#include <netdb.h>
#include <unistd.h>
#include <fcntl.h>
#include <arpa/inet.h>
#include <sys/un.h>
#include <sys/epoll.h>
#include <sys/timerfd.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/eventfd.h>
#include "network.h"


/* Set non-blocking socket */
int set_nonblocking(int fd) {
    int flags, result;
    flags = fcntl(fd, F_GETFL, 0);

    if (flags == -1)
        goto err;

    result = fcntl(fd, F_SETFL, flags | O_NONBLOCK);
    if (result == -1)
        goto err;

    return 0;

err:

    perror("set_nonblocking");
    return -1;
}

/* Disable Nagle's algorithm by setting TCP_NODELAY */
int set_tcp_nodelay(int fd) {
    return setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &(int) {1}, sizeof(int));
}


static int create_and_bind_unix(const char *sockpath) {

    struct sockaddr_un addr;
    int fd;

    if ((fd = socket(AF_UNIX, SOCK_STREAM, 0)) == -1) {
        perror("socket error");
        return -1;
    }

    memset(&addr, 0, sizeof(addr));
    addr.sun_family = AF_UNIX;

    strncpy(addr.sun_path, sockpath, sizeof(addr.sun_path) - 1);
    unlink(sockpath);

    if (bind(fd, (struct sockaddr*) &addr, sizeof(addr)) == -1) {
        perror("bind error");
        return -1;
    }

    return fd;
}


static int create_and_bind_tcp(const char *host, const char *port) {

    struct addrinfo hints = {
        .ai_family = AF_UNSPEC,
        .ai_socktype = SOCK_STREAM,
        .ai_flags = AI_PASSIVE
    };

    struct addrinfo *result, *rp;
    int sfd;

    if (getaddrinfo(host, port, &hints, &result) != 0) {
        perror("getaddrinfo error");
        return -1;
    }

    for (rp = result; rp != NULL; rp = rp->ai_next) {
        sfd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);

        if (sfd == -1) continue;

        /* set SO_REUSEADDR so the socket will be reusable after process kill */
        if (setsockopt(sfd, SOL_SOCKET, (SO_REUSEPORT | SO_REUSEADDR),
                       &(int) { 1 }, sizeof(int)) < 0)
            perror("SO_REUSEADDR");

        if ((bind(sfd, rp->ai_addr, rp->ai_addrlen)) == 0) {
            /* Succesful bind */
            break;
        }
        close(sfd);
    }

    if (rp == NULL) {
        perror("Could not bind");
        return -1;
    }

    freeaddrinfo(result);
    return sfd;
}


int create_and_bind(const char *host, const char *port, int socket_family) {

    int fd;

    if (socket_family == UNIX) {
        fd = create_and_bind_unix(host);
    } else {
        fd = create_and_bind_tcp(host, port);
    }

    return fd;
}


/*
 * Create a non-blocking socket and make it listen on the specfied address and
 * port
 */
int make_listen(const char *host, const char *port, int socket_family) {

    int sfd;

    if ((sfd = create_and_bind(host, port, socket_family)) == -1)
        abort();

    if ((set_nonblocking(sfd)) == -1)
        abort();

    // Set TCP_NODELAY only for TCP sockets
    if (socket_family == INET)
        set_tcp_nodelay(sfd);

    if ((listen(sfd, conf->tcp_backlog)) == -1) {
        perror("listen");
        abort();
    }

    return sfd;
}


int accept_connection(int serversock) {

    int clientsock;
    struct sockaddr_in addr;
    socklen_t addrlen = sizeof(addr);

    if ((clientsock = accept(serversock,
                             (struct sockaddr *) &addr, &addrlen)) < 0)
        return -1;

    set_nonblocking(clientsock);

    // Set TCP_NODELAY only for TCP sockets
    if (conf->socket_family == INET)
        set_tcp_nodelay(clientsock);

    char ip_buff[INET_ADDRSTRLEN + 1];
    if (inet_ntop(AF_INET, &addr.sin_addr,
                  ip_buff, sizeof(ip_buff)) == NULL) {
        close(clientsock);
        return -1;
    }

    return clientsock;
}

/* Send all bytes contained in buf, updating sent bytes counter */
ssize_t send_bytes(int fd, const unsigned char *buf, size_t len) {

    size_t total = 0;
    size_t bytesleft = len;
    ssize_t n = 0;

    while (total < len) {
        n = send(fd, buf + total, bytesleft, MSG_NOSIGNAL);
        if (n == -1) {
            if (errno == EAGAIN || errno == EWOULDBLOCK)
                break;
            else
                goto err;
        }
        total += n;
        bytesleft -= n;
    }

    return total;

err:

    fprintf(stderr, "send(2) - error sending data: %s", strerror(errno));
    return -1;
}

/*
 * Receive a given number of bytes on the descriptor fd, storing the stream of
 * data into a 2 Mb capped buffer
 */
ssize_t recv_bytes(int fd, unsigned char *buf, size_t bufsize) {

    ssize_t n = 0;
    ssize_t total = 0;

    while (total < (ssize_t) bufsize) {

        if ((n = recv(fd, buf, bufsize - total, 0)) < 0) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                break;
            } else
                goto err;
        }

        if (n == 0)
            return 0;

        buf += n;
        total += n;
    }

    return total;

err:

    fprintf(stderr, "recv(2) - error reading data: %s", strerror(errno));
    return -1;
}

{% endhighlight %}

## Basic closure system

To make more easy and comfortable the usage of the `EPOLL` API, being not so
complex operations to make, I built a simple abstraction on top of the
multiplexing interface to make it possible to register callback functions that
will be executed on events happening.

There's two types of callback which can be defined, the common ones, that will be
triggered with events and the periodic ones, that will be executed automatically
every tick of time interval defined. So let's wrap the epoll loop into a dedicated
structure, we'll do the same for the callback functions, defining a structure with
some fields usefull for the execution of the callback.

**src/network.h**

{% highlight c %}

/* Event loop wrapper structure, define an EPOLL loop and his status. The
 * EPOLL instance use EPOLLONESHOT for each event and must be re-armed
 * manually, in order to allow future uses on a multithreaded architecture.
 */
struct evloop {
    int epollfd;
    int max_events;
    int timeout;
    int status;
    struct epoll_event *events;
    /* Dynamic array of periodic tasks, a pair descriptor - closure */
    int periodic_maxsize;
    int periodic_nr;
    struct {
        int timerfd;
        struct closure *closure;
    } **periodic_tasks;
} evloop;


typedef void callback(struct evloop *, void *);

/*
 * Callback object, represents a callback function with an associated
 * descriptor if needed, args is a void pointer which can be a structure
 * pointing to callback parameters and closure_id is a UUID for the closure
 * itself.
 * The last two fields are payload, a serialized version of the result of
 * a callback, ready to be sent through wire and a function pointer to the
 * callback function to execute.
 */
struct closure {
    int fd;
    void *obj;
    void *args;
    char closure_id[UUID_LEN];
    struct bytestring *payload;
    callback *call;
};


struct evloop *evloop_create(int, int);

void evloop_init(struct evloop *, int, int);

void evloop_free(struct evloop *);

/*
 * Blocks in a while(1) loop awaiting for events to be raised on monitored
 * file descriptors and executing the paired callback previously registered
 */
int evloop_wait(struct evloop *);

/*
 * Register a closure with a function to be executed every time the
 * paired descriptor is re-armed.
 */
void evloop_add_callback(struct evloop *, struct closure *);

/*
 * Register a periodic closure with a function to be executed every
 * defined interval of time.
 */
void evloop_add_periodic_task(struct evloop *,
                              int,
                              unsigned long long,
                              struct closure *);

/*
 * Unregister a closure by removing the associated descriptor from the
 * EPOLL loop
 */
int evloop_del_callback(struct evloop *, struct closure *);

/*
 * Rearm the file descriptor associated with a closure for read action,
 * making the event loop to monitor the callback for reading events
 */
int evloop_rearm_callback_read(struct evloop *, struct closure *);

/*
 * Rearm the file descriptor associated with a closure for write action,
 * making the event loop to monitor the callback for writing events
 */
int evloop_rearm_callback_write(struct evloop *, struct closure *);

/* Epoll management functions */
int epoll_add(int, int, int, void *);

/*
 * Modify an epoll-monitored descriptor, automatically set EPOLLONESHOT in
 * addition to the other flags, which can be EPOLLIN for read and EPOLLOUT for
 * write
 */
int epoll_mod(int, int, int, void *);

/*
 * Remove a descriptor from an epoll descriptor, making it no-longer monitored
 * for events
 */
int epoll_del(int, int);


{% endhighlight %}

After some declarations on the header for network utility we can pass to the
implementation of the functions.

We start with simple creation, init and deletion of the previously declared
structure `evloop`, consisting in a file descriptor for the epoll loop, a
number of events to monitor, a timeout in milliseconds that defines the time
we'll block the loop, the status of the loop (will probably contain error codes
for faulting cases) and finally a dynamic array of periodic tasks that will be
executed.

**src/network.c**

{% highlight c %}

/******************************
 *         EPOLL APIS         *
 ******************************/


#define EVLOOP_INITIAL_SIZE 4


struct evloop *evloop_create(int max_events, int timeout) {

    struct evloop *loop = malloc(sizeof(*loop));

    evloop_init(loop, max_events, timeout);

    return loop;
}


void evloop_init(struct evloop *loop, int max_events, int timeout) {
    loop->max_events = max_events;
    loop->events = malloc(sizeof(struct epoll_event) * max_events);
    loop->epollfd = epoll_create1(0);
    loop->timeout = timeout;
    loop->periodic_maxsize = EVLOOP_INITIAL_SIZE;
    loop->periodic_nr = 0;
    loop->periodic_tasks =
        malloc(EVLOOP_INITIAL_SIZE * sizeof(*loop->periodic_tasks));
    loop->status = 0;
}


void evloop_free(struct evloop *loop) {
    free(loop->events);
    for (int i = 0; i < loop->periodic_nr; i++)
        free(loop->periodic_tasks[i]);
    free(loop->periodic_tasks);
    free(loop);
}

{% endhighlight %}

Now, epoll API is extensively documentated on its man page, but we'll need 3
functions to add, remove and modify monitored descriptors and trigger events,
using `EPOLLET` flag, in order to use epoll on edge-trigger and avoid in a
future multithreaded implementation to wake up all threads at once every time a
new event is triggered one or more descriptor are ready to read or write, but this
is another story, also this explained clearly on the man page.

**src/network.c**

{% highlight c %}

int epoll_add(int efd, int fd, int evs, void *data) {

    struct epoll_event ev;
    ev.data.fd = fd;

    // Being ev.data a union, in case of data != NULL, fd will be set to random
    if (data)
        ev.data.ptr = data;

    ev.events = evs | EPOLLET | EPOLLONESHOT;

    return epoll_ctl(efd, EPOLL_CTL_ADD, fd, &ev);
}


int epoll_mod(int efd, int fd, int evs, void *data) {

    struct epoll_event ev;
    ev.data.fd = fd;

    // Being ev.data a union, in case of data != NULL, fd will be set to random
    if (data)
        ev.data.ptr = data;

    ev.events = evs | EPOLLET | EPOLLONESHOT;

    return epoll_ctl(efd, EPOLL_CTL_MOD, fd, &ev);
}


int epoll_del(int efd, int fd) {
    return epoll_ctl(efd, EPOLL_CTL_DEL, fd, NULL);
}

{% endhighlight %}

Two things to be noted:

- First, the main structure `epoll_event` contains a `union` inside, which
  accept a file descriptor or a `void *` pointer. We'll use the latter, this
  way we'll be able to pass around more informations and use our custom
  closure, the file descriptor will be stored inside the structure pointed.

- Second, our two add and mod functions accepts as third parameters a set of
  events, mostly `EPOLLIN` or `EPOLLOUT`, but they add `EPOLLONESHOT` to them.
  This way every time an event is triggered, the descriptor must be manually
  rearmed for read or write events.This is done to maintain some degree of
  control on low level events triggering and to left an open door in case of
  future multithreaded implementation, [this great
  article](https://idea.popcount.org/2017-02-20-epoll-is-fundamentally-broken-12/)
  explains wonderfully the advantages (or the broken parts) of the epoll and
  why it's better to use `EPOLLONESHOT` flag.

We move forward now to implement the basic closure system and the wait loop for
read and write events, as well as periodic timed callbacks.

**src/network.c**

{% highlight c %}

void evloop_add_callback(struct evloop *loop, struct closure *cb) {
    if (epoll_add(loop->epollfd, cb->fd, EPOLLIN, cb) < 0)
        perror("Epoll register callback: ");
}


void evloop_add_periodic_task(struct evloop *loop,
                              int seconds,
                              unsigned long long ns,
                              struct closure *cb) {

    struct itimerspec timervalue;

    int timerfd = timerfd_create(CLOCK_MONOTONIC, 0);

    memset(&timervalue, 0x00, sizeof(timervalue));

    // Set initial expire time and periodic interval
    timervalue.it_value.tv_sec = seconds;
    timervalue.it_value.tv_nsec = ns;
    timervalue.it_interval.tv_sec = seconds;
    timervalue.it_interval.tv_nsec = ns;

    if (timerfd_settime(timerfd, 0, &timervalue, NULL) < 0) {
        perror("timerfd_settime");
        return;
    }

    // Add the timer to the event loop
    struct epoll_event ev;
    ev.data.fd = timerfd;
    ev.events = EPOLLIN;

    if (epoll_ctl(loop->epollfd, EPOLL_CTL_ADD, timerfd, &ev) < 0) {
        perror("epoll_ctl(2): EPOLLIN");
        return;
    }

    /* Store it into the event loop */
    if (loop->periodic_nr + 1 > loop->periodic_maxsize) {
        loop->periodic_maxsize *= 2;
        loop->periodic_tasks =
            realloc(loop->periodic_tasks,
                    loop->periodic_maxsize * sizeof(*loop->periodic_tasks));
    }

    loop->periodic_tasks[loop->periodic_nr] =
        malloc(sizeof(*loop->periodic_tasks[loop->periodic_nr]));

    loop->periodic_tasks[loop->periodic_nr]->closure = cb;
    loop->periodic_tasks[loop->periodic_nr]->timerfd = timerfd;
    loop->periodic_nr++;

}


int evloop_wait(struct evloop *el) {

    int rc = 0;
    int events = 0;
    long int timer = 0L;
    int periodic_done = 0;

    while (1) {

        events = epoll_wait(el->epollfd, el->events,
                            el->max_events, el->timeout);

        if (events < 0) {

            /* Signals to all threads. Ignore it for now */
            if (errno == EINTR)
                continue;

            /* Error occured, break the loop */
            rc = -1;
            el->status = errno;
            break;
        }

        for (int i = 0; i < events; i++) {

            /* Check for errors */
            if ((el->events[i].events & EPOLLERR) ||
                (el->events[i].events & EPOLLHUP) ||
                (!(el->events[i].events & EPOLLIN) &&
                 !(el->events[i].events & EPOLLOUT))) {

                /* An error has occured on this fd, or the socket is not
                   ready for reading, closing connection */
                perror ("epoll_wait(2)");
                shutdown(el->events[i].data.fd, 0);
                close(el->events[i].data.fd);
                el->status = errno;
                continue;
            }

            struct closure *closure = el->events[i].data.ptr;
            periodic_done = 0;

            for (int i = 0; i < el->periodic_nr && periodic_done == 0; i++) {
                if (el->events[i].data.fd == el->periodic_tasks[i]->timerfd) {
                    struct closure *c = el->periodic_tasks[i]->closure;
                    (void) read(el->events[i].data.fd, &timer, 8);
                    c->call(el, c->args);
                    periodic_done = 1;
                }
            }

            if (periodic_done == 1)
                continue;

            /* No error events, proceed to run callback */
            closure->call(el, closure->args);
        }
    }

    return rc;
}


int evloop_rearm_callback_read(struct evloop *el, struct closure *cb) {
    return epoll_mod(el->epollfd, cb->fd, EPOLLIN, cb);
}


int evloop_rearm_callback_write(struct evloop *el, struct closure *cb) {
    return epoll_mod(el->epollfd, cb->fd, EPOLLOUT, cb);
}


int evloop_del_callback(struct evloop *el, struct closure *cb) {
    return epoll_del(el->epollfd, cb->fd);
}

{% endhighlight %}

{% highlight bash %}
sol/
 ├── src/
 │    ├── mqtt.h
 |    ├── mqtt.c
 │    ├── network.h
 │    ├── network.c
 │    ├── pack.h
 │    └── pack.c
 ├── CHANGELOG
 ├── CMakeLists.txt
 ├── COPYING
 └── README.md
{% endhighlight %}
