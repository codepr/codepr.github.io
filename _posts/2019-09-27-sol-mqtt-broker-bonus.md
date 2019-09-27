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
wasn't aware of, the **spinlock**, so I promptly browsed the pthread
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
rest can be safely applied by merging the *master* branch into the *tutorial* one.
