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
    if (DISCONNECT_TYPE < (*header >> 4) || CONNECT_TYPE > (*header >> 4))
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
    unsigned char *dest = sol_malloc(len + 1);
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
    unsigned char *str = sol_malloc(len);
    memcpy(str, init, len);
    return str;
}

/* Same as bstring_copy but setting the entire content of the string to 0 */
bstring bstring_empty(size_t len) {
    unsigned char *str = sol_malloc(len);
    memset(str, 0x00, len);
    return str;
}

void bstring_destroy(bstring s) {
    /*
     * Being allocated with utils custom functions just free it with the
     * corrispective free function
     */
    sol_free(s);
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
    eventfd_t io_event;
    struct sol_client *client;
    bstring reply;
    union mqtt_packet *payload;
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
