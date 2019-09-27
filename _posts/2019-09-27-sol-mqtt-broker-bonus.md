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
to see if this approach was feasible and advisable in terms of performances.
Some time ago I stumbled upon an article mentioning a type of mutex which I wasn't
aware of, the *spinlock*, so I promptly browsed the pthread documentation and it
turned out that a spinlock is essentially a normal mutex, but behave slightly
differently, in fact, with a normal mutex, when a thread acquires lock to e.g.
accessing a shared resource, the CPU puts all other threads needing to access that
critical part of code to sleep, waking them up only after the resource guarded
by the mutex object has been released; a spinlock instead let other threads to
constantly try to access the critical section till the locked part is released, in
a sort of busy wait state. This can have some benefits and drawbacks depending on the
use case, sometimes maybe the shared resources are held for so little time that it is
more costly to put to sleep and wake up all other threads than to let them try
to access till the lock is released, on the other side, this approach wastes lot of
CPU cycles, especially in the case of a longer than expected lock on the resource.

So the main intuition was that in *Sol* as of now, there're not significant CPU-bound
parts, the main bottleneck is represented by the network communication and the
datastructures which are essentially the shared sections on the systems are fast
enough to outrun each TCP transaction.
