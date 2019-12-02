---
layout: post
title: "Sol - An MQTT broker from scratch. Bonus - Multithreading"
description: "Writing an MQTT broker from scratch, to really understand something you have to build it."
categories: c unix tutorial epoll
---

So in the previous 6 parts we explored a fair amount of common CS topics such
as networks and data structures, the little journey ended up with a bugged but
working toy to play with.
<!--more-->
Out of curiosity I decided to try and make a dangerous step forward (or
backward..probably lateral) and modify the server part of the project to be
multithreaded, mainly to better understand posix threads working mechanism and
to see if this approach was feasible and advisable in terms of performance.

Some time ago I stumbled upon an article mentioning a type of mutex which I
wasn't aware of, the `spinlock`, so I promptly browsed the pthread
documentation and it turned out that a spinlock is essentially a normal mutex,
but behave slightly differently, in fact, with a normal mutex, when a thread
acquires lock to e.g. accessing a shared resource, the CPU puts all other
threads needing to access that critical part of code to sleep, waking them up
only after the resource guarded by the mutex object has been released; a
spinlock instead let other threads to constantly try to access the critical
section till the locked part is released, in a sort of busy wait state. This
can have some benefits and drawbacks depending on the use case, sometimes maybe
the shared resources are held for so little time that it is more costly to put
to sleep and wake up all other threads than to let them try to access till the
lock is released, on the other side, this approach wastes lot of CPU cycles,
especially in the case of a longer than expected lock on the resource.

So the main intuition was that in **Sol** as of now, there're not significant
CPU-bound parts, the main bottleneck is represented by the network
communication and the datastructures which are essentially the shared sections
on the systems are fast enough to outrun each TCP transaction.

### Refactoring and bug hunting

Before running into this bloody run I thought it would be a better idea to
adjust some clunky parts and fix some of the probably unseen bugs I introduced
a commit at a time during the first draft.
The first attempts involved some in-place refactoring on a development branch
just to test the waters and make an idea of the difficulties I was going to
encounter. Initially I tried to integrate the concurrency parts with the
closure APIs we've seen in [part 2](../sol-mqtt-broker-p2) but it turned out to
be harder than I thought and I didn't like the idea of  shared EPOLL with
multiple threads that do everything.

Eventually I realized that a common approach I already experimented in another
project was the best one, but involved some heavy refactoring, including the
removal of the said closure system (you know, it's still increment even by
removing features, they're the best increments :P). The main idea was to
instantiate two distinct threadpool, or better, a mandatory one to handle IO
and another which could also be a single thread to handle the work parts like
command handling and storing of informations.<br/>

{% highlight bash %}

       MAIN                  1...N                  1...N

      [EPOLL]             [IO EPOLL]             [WORK EPOLL]
   ACCEPT THREAD        IO THREAD POOL        WORKER THREAD POOL
   -------------        --------------        ------------------
         |                     |                      |
       ACCEPT                  |                      |
         | ------------------> |                      |
         |              READ AND DECODE               |
         |                     | -------------------> |
         |                     |                     WORK
         |                     | <------------------- |
         |                   WRITE                    |
         |                     |                      |
       ACCEPT                  |                      |
         | ------------------> |                      |

{% endhighlight %}

As shown above, we'll have 3 epoll instances:

- a single thread exclusively used to accept connections, can be the main thread
- 2 or more threads dedicated to I/O
- 1 or more threads dedicated to command handling

A nice thing that happened is that this process managed to elicit lot of the
bugs I mentioned before, and forced improvements on some fragile parts, like
the data stream receptions and parsing of instructions.
I won't walk through all the refactoring process, it would be deadly boring,
I'll just enlight some of the most important parts that needed adjustements, the
rest can be safely applied by merging the `master` branch into the `tutorial` one.

#### Packets fragmentation is not funny

The first and foremost aspect to check was the network communication, by mainly
testing in local I only noticed after some heavier benchmarking that sometimes
the system was losing some packets, or better, the kernel buffer was probably
flooded and started to fragment some payloads, TCP is after all a stream protocol
and it's perfectly fine to segment data during sending. It was a little naive by me
to not handle this properly initially, anyway this led to some nasty behaviours,
like packets wrongly parsed, being the first byte of incoming chunk of data, recognized
as a different than expected instruction.<br/>
There also was some stupid overflow related oversights, like using an `unsigned
short` to handle a field which was supposed to be longer than 65535, like the
size of the entire command which could be well over that limit (2 MB). So one
of the most important fix was the `recv_packet` function on the **server.c**
module, specifically the addition of a loop tracking the remaning to read
bytes, which ensure we read the entire packet in a single call:

{% highlight c %}
ssize_t recv_packet(int clientfd, unsigned char **buf, unsigned char *header) {
    ssize_t nbytes = 0;
    unsigned char *tmpbuf = *buf;
    /* Read the first byte, it should contain the message type code */
    if ((nbytes = recv_bytes(clientfd, *buf, 4)) <= 0)
        return -ERRCLIENTDC;
    *header = *tmpbuf;
    tmpbuf++;
    /* Check for OPCODE, if an unknown OPCODE is received return an error */
    if (DISCONNECT < (*header >> 4) || CONNECT > (*header >> 4))
        return -ERRPACKETERR;
    /*
     * Read remaning length bytes which starts at byte 2 and can be long to 4
     * bytes based on the size stored, so byte 2-5 is dedicated to the packet
     * length.
     */
    int n = 0;
    unsigned pos = 0;
    unsigned long long tlen = mqtt_decode_length(&tmpbuf, &pos);
    /*
     * Set return code to -ERRMAXREQSIZE in case the total packet len exceeds
     * the configuration limit `max_request_size`
     */
    if (tlen > conf->max_request_size) {
        nbytes = -ERRMAXREQSIZE;
        goto exit;
    }
    if (tlen <= 4)
        goto exit;
    int offset = 4 - pos -1;
    unsigned long long remaining_bytes = tlen - offset;
    /* Read remaining bytes to complete the packet */
    while (remaining_bytes > 0) {
        if ((n = recv_bytes(clientfd, tmpbuf + offset, remaining_bytes)) < 0)
            goto err;
        remaining_bytes -= n;
        nbytes += n;
        offset += n;
    }
    nbytes -= (pos + 1);
exit:
    *buf += pos + 1;
    return nbytes;
err:
    close(clientfd);
    return nbytes;
}
{% endhighlight %}

Another good improvement was the correction of the packing and unpacking
functions (thanks to [beej networking
guide](https://beej.us/guide/bgnet/html/single/bgnet.html#serialization), this
guide is pure gold) and the addition of some helper functions to handle
integer and bytes unpacking:

**pack.c**

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
            val = unpacku16(*buf);
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

And finally it's worth the mention the use of useful enhanced strings, instead
of using a structure to handle them, we piggyback the size of every array of chars
to each malloc'ed string, this way, with some helper functions, this bytestrings
can be used as normal strings as well.

{% highlight c %}
/*
 * Return the length of the string without having to call strlen, thus this
 * works also with non-nul terminated string. The length of the string is in
 * fact stored in memory in an unsigned long just before the position of the
 * string itself.
 */
size_t bstring_len(const bstring s) {
    return *((size_t *) (s - sizeof(size_t)));
}

bstring bstring_new(const unsigned char *init) {
    if (!init)
        return NULL;
    size_t len = strlen((const char *) init);
    return bstring_copy(init, len);
}

bstring bstring_copy(const unsigned char *init, size_t len) {
    /*
     * The strategy would be to piggyback the real string to its stored length
     * in memory, having already implemented this logic before to actually
     * track memory usage of the system, we just need to malloc it with the
     * custom malloc in utils
     */
    unsigned char *str = malloc(len);
    memcpy(str, init, len);
    return str;
}

/* Same as bstring_copy but setting the entire content of the string to 0 */
bstring bstring_empty(size_t len) {
    unsigned char *str = malloc(len);
    memset(str, 0x00, len);
    return str;
}

void bstring_destroy(bstring s) {
    /*
     * Being allocated with utils custom functions just free it with the
     * corrispective free function
     */
    free(s);
}
{% endhighlight %}

### Make the accept, IO and work parts communicate

Epoll shared between threads presents some traps, and the main debate is to
wether use an epoll descriptor for each thread or share a single descriptor
between multiple threads. I personally prefer the second approach, in order to
let the kernel do the load-balancing, epoll APIs is in-fact threadsafe, while
the first one, thus being most of the time easier, gave up a bit of control on
clients handling, thus every thread will have a subset of connections and work
to handle, without any warranty that some threads could be starved and some
other being over heavy load.<br/>
The main issues to address are the thundering herd problem, communication and
of course handling of shared resource on the business logic sections. The first
problem should be tackled by the `EPOLLONESHOT` flag, in order to wake up just
one thread per time and manually rearm the descriptor for read/write events.
For the shared resources we opted for the `spinlock` solution on critical parts,
the communications remain to be solved.

Queues are usually the go-to solutions in these cases, but instead of writing a
threadsafe queue which could prove tricky, especially if our purpose is to
improve performance, it was better to scratch a bit the surface and search for
a lighweight already implemented solution. It came in the form of event handling
at a kernel level. Running `man eventfd` I found the answers i seeked for, eventfd
could be used to fire arbitrary events to different epoll descriptor, just registering
one to different epoll descriptor, while looping waiting for events to be triggered
they can be used to effectively communicate and share data safely between them.
First I needed two structures to wrap and handle events and one to instantiate the
involved epoll descriptors.

{% highlight c %}
/*
 * IO event strucuture, it's the main information that will be communicated
 * between threads, every request packet will be wrapped into an IO event and
 * passed to the work EPOLL, in order to be handled by the worker thread pool.
 * Then finally, after the execution of the command, it will be updated and
 * passed back to the IO epoll loop to be written back to the requesting client
 */
struct io_event {
    int epollfd;
    eventfd_t eventfd;
    bstring reply;
    struct sol_client *client;
    union mqtt_packet *data;
};

/*
 * Shared epoll object, contains the IO epoll and Worker epoll descriptors,
 * as well as the server descriptor and the timer fd for repeated routines.
 * Each thread will receive a copy of a pointer to this structure, to have
 * access to all file descriptor running the application
 */
struct epoll {
    int io_epollfd;
    int w_epollfd;
    int serverfd;
    int expirefd;
    int busfd;
};
{% endhighlight %}

These two structures enable the communication between threads
- `io_event` represents a I/O event, a wrapper around an interaction from a
  connected client, it'll be the object that will be passed in as arguments to
  all handler callbacks as a pointer, its content is mostly self-explanatory,
  the epollfd is the one of the epoll instance where the fd "belongs", which
  will be an IO thread epoll one, there's a reference to the source client of
  the request, a bystestring for the reply, a payload and the `eventfd_t` field
  which represents our trigger to wake up a worker epoll thread.
- `epoll` is more of a `const` structure inited in the main entry point function
  `start_server` passed around to the `accept_loop` function on the main thread,
  to `io_worker` threads and finally to the `worker_thread` threads.
  It store the epoll descriptor of the IO dedicated threadpool, and the one of
  the worker threadpool, while `busfd` is still unused, it'll eventually come
  useful later if I'll be crazy enough to try to scale-out the system across a
  cluster of machines, implementing some sort of gossip protocol to share
  informations, but that's another whole world to explore.

So we'll end up init the `epoll` struct on `start_server` like this

**server.c**

{% highlight c %}
int start_server(const char *addr, const char *port) {
    // init logics here
    ...

    /* Start listening for new connections */
    int sfd = make_listen(addr, port, conf->socket_family);

    struct epoll epoll = {
        .io_epollfd = epoll_create1(0),
        .w_epollfd = epoll_create1(0),
        .serverfd = sfd,
        .busfd = -1  // unused
    };

    /* Start the expiration keys check routine */
    struct itimerspec timervalue;

    memset(&timervalue, 0x00, sizeof(timervalue));

    timervalue.it_value.tv_sec = conf->stats_pub_interval;
    timervalue.it_value.tv_nsec = 0;
    timervalue.it_interval.tv_sec = conf->stats_pub_interval;
    timervalue.it_interval.tv_nsec = 0;

    // add expiration keys cron task
    int exptimerfd = add_cron_task(epoll.w_epollfd, &timervalue);

    epoll.expirefd = exptimerfd;

    /*
     * We need to watch for global eventfd in order to gracefully shutdown IO
     * thread pool and worker pool
     */
    epoll_add(epoll.io_epollfd, conf->run, EPOLLIN, NULL);
    epoll_add(epoll.w_epollfd, conf->run, EPOLLIN, NULL);

    pthread_t iothreads[IOPOOLSIZE];
    pthread_t workers[WORKERPOOLSIZE];

    /* Start I/O thread pool */

    for (int i = 0; i < IOPOOLSIZE; ++i)
        pthread_create(&iothreads[i], NULL, &io_worker, &epoll);

    /* Start Worker thread pool */

    for (int i = 0; i < WORKERPOOLSIZE; ++i)
        pthread_create(&workers[i], NULL, &worker, &epoll);

    sol_info("Server start");
    info.start_time = time(NULL);

    // Main thread for accept new connections
    accept_loop(&epoll);
    // resources release and clean out here
    ...
}
{% endhighlight %}

All we have to do is run an accept loop where all incoming connections will be
handled, a new `struct client` will be instantiated with partial init (some fields
could be modified later with commands, like the `connect` one) and the resulting
connected descriptor wrapped in the `client` will be associated to the IO epoll
descriptor, that's it, it's an IO worker problem now.

{% highlight c %}
static void accept_loop(struct epoll *epoll) {
    int events = 0;
    struct epoll_event *e_events =
        malloc(sizeof(struct epoll_event) * EPOLL_MAX_EVENTS);
    int epollfd = epoll_create1(0);
    /*
     * We want to watch for events incoming on the server descriptor (e.g. new
     * connections)
     */
    epoll_add(epollfd, epoll->serverfd, EPOLLIN | EPOLLONESHOT, NULL);
    /*
     * And also to the global event fd, this one is useful to gracefully
     * interrupt polling and thread execution
     */
    epoll_add(epollfd, conf->run, EPOLLIN, NULL);
    while (1) {
        events = epoll_wait(epollfd, e_events, EPOLL_MAX_EVENTS, EPOLL_TIMEOUT);
        if (events < 0) {
            /* Signals to all threads. Ignore it for now */
            if (errno == EINTR)
                continue;
            /* Error occured, break the loop */
            break;
        }
        for (int i = 0; i < events; ++i) {
            /* Check for errors */
            EPOLL_ERR(e_events[i]) {
                /*
                 * An error has occured on this fd, or the socket is not
                 * ready for reading, closing connection
                 */
                perror("accept_loop :: epoll_wait(2)");
                close(e_events[i].data.fd);
            } else if (e_events[i].data.fd == conf->run) {
                /* And quit event after that */
                eventfd_t val;
                eventfd_read(conf->run, &val);
                sol_debug("Stopping epoll loop. Thread %p exiting.",
                          (void *) pthread_self());
                goto exit;
            } else if (e_events[i].data.fd == epoll->serverfd) {
                while (1) {
                    /*
                     * Accept a new incoming connection assigning ip address
                     * and socket descriptor to the connection structure
                     * pointer passed as argument
                     */
                    int fd = accept_connection(epoll->serverfd);
                    if (fd < 0)
                        break;
                    /*
                     * Create a client structure to handle his context
                     * connection
                     */
                    struct sol_client *client = malloc(sizeof(*client));
                    if (!client)
                        return;
                    /* Populate client structure */
                    client->fd = fd;
                    /* Record last action as of now */
                    client->last_action_time = time(NULL);
                    /* Add it to the epoll loop */
                    epoll_add(epoll->io_epollfd, fd,
                              EPOLLIN | EPOLLONESHOT, client);
                    /* Rearm server fd to accept new connections */
                    epoll_mod(epollfd, epoll->serverfd, EPOLLIN, NULL);
                    /* Record the new client connected */
                    info.nclients++;
                    info.nconnections++;
                }
            }
        }
    }
exit:
    free(e_events);
}
{% endhighlight %}

From now on the IO worker will just handle, as the name explain, the IO
operations of the connected client, like new instructions sent or output to be
sent out after a subscription request for example.

{% highlight c %}
static void *io_worker(void *arg) {
    struct epoll *epoll = arg;
    int events = 0;
    ssize_t sent = 0;
    struct epoll_event *e_events =
        malloc(sizeof(struct epoll_event) * EPOLL_MAX_EVENTS);
    /* Raw bytes buffer to handle input from client */
    unsigned char *buffer = malloc(conf->max_request_size);
    while (1) {
        events = epoll_wait(epoll->io_epollfd, e_events,
                            EPOLL_MAX_EVENTS, EPOLL_TIMEOUT);
        if (events < 0) {
            /* Signals to all threads. Ignore it for now */
            if (errno == EINTR)
                continue;
            /* Error occured, break the loop */
            break;
        }
        for (int i = 0; i < events; ++i) {
            /* Check for errors */
            EPOLL_ERR(e_events[i]) {
                /* An error has occured on this fd, or the socket is not
                   ready for reading, closing connection */
                perror("io_worker :: epoll_wait(2)");
                close(e_events[i].data.fd);
            } else if (e_events[i].data.fd == conf->run) {
                /* And quit event after that */
                eventfd_t val;
                eventfd_read(conf->run, &val);
                sol_debug("Stopping epoll loop. Thread %p exiting.",
                          (void *) pthread_self());
                goto exit;
            } else if (e_events[i].events & EPOLLIN) {
                struct io_event *event = malloc(sizeof(*event));
                event->epollfd = epoll->io_epollfd;
                event->data = malloc(sizeof(*event->data));
                event->client = e_events[i].data.ptr;
                /*
                 * Received a bunch of data from a client, after the creation
                 * of an IO event we need to read the bytes and encoding the
                 * content according to the protocol
                 */
                int rc = read_data(event->client->fd, buffer, event->data); // here we essentially call recv_packet
                if (rc == 0) {
                    /*
                     * All is ok, raise an event to the worker poll EPOLL and
                     * link it with the IO event containing the decode payload
                     * ready to be processed
                     */
                    eventfd_t ev = eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK);
                    event->eventfd = ev;
                    epoll_add(epoll->w_epollfd, ev,
                              EPOLLIN | EPOLLONESHOT, event);
                    /* Record last action as of now */
                    event->client->last_action_time = time(NULL);
                    /* Fire an event toward the worker thread pool */
                    eventfd_write(ev, 1);
                } else if (rc == -ERRCLIENTDC || rc == -ERRPACKETERR) {
                    /*
                     * We got an unexpected error or a disconnection from the
                     * client side, remove client from the global map and
                     * free resources allocated such as io_event structure and
                     * paired payload
                     */
                    close(event->client->fd);
                    hashtable_del(sol.clients, event->client->client_id);
                    free(event->data);
                    free(event);
                }
            } else if (e_events[i].events & EPOLLOUT) {
                struct io_event *event = e_events[i].data.ptr;
                /*
                 * Write out to client, after a request has been processed in
                 * worker thread routine. Just send out all bytes stored in the
                 * reply buffer to the reply file descriptor.
                 */
                if ((sent = send_bytes(event->client->fd,
                                       event->reply,
                                       bstring_len(event->reply))) < 0) {
                    close(event->client->fd);
                } else {
                    /*
                     * Rearm descriptor, we're using EPOLLONESHOT feature to avoid
                     * race condition and thundering herd issues on multithreaded
                     * EPOLL
                     */
                    epoll_mod(epoll->io_epollfd,
                              event->client->fd, EPOLLIN, event->client);
                }
                // Update information stats
                info.bytes_sent += sent < 0 ? 0 : sent;
                /* Free resource, ACKs will be free'd closing the server */
                bstring_destroy(event->reply);
                mqtt_packet_release(event->data, event->data->header.bits.type);
                close(event->eventfd);
                free(event);
            }
        }
    }
exit:
    free(e_events);
    free(buffer);
    return NULL;
}
{% endhighlight %}

As we can see, inside the `EPOLLIN if` branch we fire a new event to the target
epoll instance, this case pointed by the worker epoll descriptor and we pass
the control over the parsed packet to the worker pool, which will apply the
business logic calling the right callback based on the received command including
error handling.<br/> To be noted that the event is created directly in the
`if` for each incoming event and closed right after the `EPOLLOUT` is triggered
for each client. Essentially it has the lifespan of a request-response.

{% highlight c %}
static void *worker(void *arg) {
    struct epoll *epoll = arg;
    int events = 0;
    long int timers = 0;
    eventfd_t val;
    struct epoll_event *e_events =
        malloc(sizeof(struct epoll_event) * EPOLL_MAX_EVENTS);
    while (1) {
        events = epoll_wait(epoll->w_epollfd, e_events,
                            EPOLL_MAX_EVENTS, EPOLL_TIMEOUT);
        if (events < 0) {
            /* Signals to all threads. Ignore it for now */
            if (errno == EINTR)
                continue;
            /* Error occured, break the loop */
            break;
        }
        for (int i = 0; i < events; ++i) {
            /* Check for errors */
            EPOLL_ERR(e_events[i]) {
                /*
                 * An error has occured on this fd, or the socket is not
                 * ready for reading, closing connection
                 */
                perror("worker :: epoll_wait(2)");
                close(e_events[i].data.fd);
            } else if (e_events[i].data.fd == conf->run) {
                /* And quit event after that */
                eventfd_read(conf->run, &val);
                sol_debug("Stopping epoll loop. Thread %p exiting.",
                          (void *) pthread_self());
                goto exit;
            } else if (e_events[i].data.fd == epoll->expirefd) {
                (void) read(e_events[i].data.fd, &timers, sizeof(timers));
                // Check for keys about to expire out
                publish_stats();
            } else if (e_events[i].events & EPOLLIN) {
                struct io_event *event = e_events[i].data.ptr;
                eventfd_read(event->eventfd, &val);
                // TODO free client and remove it from the global map in case
                // of QUIT command (check return code)
                int reply = handlers[event->data->header.bits.type](event);
                if (reply == REPLY)
                    epoll_mod(event->epollfd, event->client->fd, EPOLLOUT, event);
                else if (reply != CLIENTDC) {
                    epoll_mod(epoll->io_epollfd,
                              event->client->fd, EPOLLIN, event->client);
                    close(event->eventfd);
                }
            }
        }
    }
exit:
    free(e_events);
    return NULL;
}
{% endhighlight %}

Commands are handled through a dispatch table, a common pattern used in C where
we map function pointers inside an array, in this case each position in the
array corresponds to an MQTT command.

This of course is only a fraction of what the ordeal has been but eventually I
came up with a somewhat working prototype, the next step will be to stress test
it a bit and see how it goes compared to the battle-tested and indisputably
better pieces of software like Mosquitto or Mosca; messing around with the
threads number and the mutex type. Lot of missing features still, no auth, no
SSL/TLS communication, no session recovery and QoS 2 handling (started some
basic work here) but the mere pub/sub part should be testable. Hopefully, this
tutorial would work as a starting point for something neater and carefully
designed.
Cya.
