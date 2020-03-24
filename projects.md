---
layout: default
title: Projects
description: "Personal pet projects, mostly learning experiences. As a side note, these are just a part of all the code I produced in the last years, being it for a significant part experimental work."
permalink: /projects/
---

### llb - Dead simple event-driven load balancer
--------------------------------------------------------------------------
(**L**)ittle(**L**)oad(**B**)alancer, a dead simple event-driven load-balancer
(< 2000 sloc).
Supports Linux (and arguably OSX) through epoll and poll/select (kqueue on
BSD-like) as fallback, it uses an event-loop library borrowed from
[Sol](https://github.com/codepr/sol.git){:target="_blank"}.
It currently supports a bunch of most common balancing algorithms like
round-robin, weighted round-robin, leastconn, hash-balancing and of course
random-balancing.

Written out of boredom/learning pupose (50/50) during self-isolation. Sure
thing there will be bugs and plenty of corner cases to be addressed.

[link](https://github.com/codepr/llb.git){:target="_blank"}

### Sol - Lightweight MQTT broker
----------------------------------------------------------------------
Oversimplified MQTT broker written from scratch, which mimick mosquitto
features. Implemented for learning how the protocol works, for now it supports
almost all MQTT v3.1.1 commands on linux platform; under linux it relies on
epoll interface introduced on kernel 2.5.44, the other fallbacks are poll/select
or kqueue on BSD-like systems.

[link](https://github.com/codepr/sol.git){:target="_blank"}

### EV - Lightweight event-loop library based on multiplexing IO
--------------------------------------------------------------------------

Light event-loop library loosely inspired by the excellent libuv, in a single
small (< 1000 sloc) header, based on the common IO multiplexing implementations
available, epoll on linux, kqueue on BSD-like and OSX, poll/select as a
fallback, dependencies-free.
Extracted and improved from another project (sol).

[link](https://github.com/codepr/ev.git){:target="_blank"}

### Narwhal - PoC of a simple continuous integration system
--------------------------------------------------------------------------

A very simple CI system, consisting in 2 main components:

- The dispatcher, a simple RESTful server exposing some APIs to submit commits
  and register new runners

- The Runner, a RESTful server as well that manage a pool of pre-allocated
  containers to run tests (and arguably other instructions in the future)
  safely inside an isolated environment.

Ideally a bunch of runners should be spread on a peer's subnet with similar
hw and each one registers itself to the dispatcher.
Beside registering itself, another way could very well be to use a
load-balancer or a proxy, registering it's URL to the dispatcher and demanding
the job distributions to it.

First project in Go, actually made to learn the language as it offers a lot of
space for improvements and incremental addition of features.

[link](https://github.com/codepr/narwhal.git){:target="_blank"}

### Tasq - distributed task queue
---------------------------------------------------------------------
Very simple distributed Task queue that allow the scheduling of job
functions to be executed on local or remote workers. Can be seen as a Proof of
Concept leveraging ZMQ sockets and cloudpickle serialization capabilities as
well as a very basic actor system to handle different loads of work from
connecting clients. Extending the codebase to support common patterns with
Redis or RabbitMQ as queue middlewares.

[link](https://github.com/codepr/tasq.git){:target="_blank"}

### TrieDB - kv store based on a trie data structure
-------------------------------------------------------------------------
Multi-threaded Key-value store based on a Trie data structure. Trie is a kind
of trees in which each node is a prefix for a key, the node position define the
keys and the associated values are set on the last node of each key. They
provide a big-O runtime complexity of O(m) on worst case, for insertion and
lookup, where m is the length of the key. The main advantage is the possibility
to query the tree by prefix, executing range scans in an easy way.

Almost all commands supported has a "prefix" version which apply the command
itself on a prefix instead of a full key.

[link](https://github.com/codepr/triedb.git){:target="_blank"}

### Aiotunnel - HTTP(S) tunneling for local and remote port-forwarding
-------------------------------------------------------------------------------
Yet another HTTP tunnel, supports two modes; a direct one which open a local
port on the host machine and redirect all TCP data to the remote side of the
tunnel, which actually connect to the desired URL. A second one which require
the client part to be run on the target system we want to expose, the server
side on a (arguably) public machine (e.g. an AWS EC2) which expose a port to
communicate to our target system through HTTP.

[link](https://github.com/codepr/aiotunnel.git){:target="_blank"}

### Orestes - Basic kv store in haskell
--------------------------------------------------------------------------
Simple implementation of a distributed key-value server, aimed to learn basic
concepts of functional programming through Haskell. For the distribution it
uses cloud-haskell libraries, based on asynchronous message protocol like
Erlang distribution actor model. It works on both a cluster of machines or on a
single one according to a master-slave topology. It currently support just the
common operations `PUT`, `GET` and `DEL` and it lacks a suitable communication
protocol, on the other side it is perfectly usable by using a generic TCP
client like Telnet or Netcat issuing commands as strings.

[link](https://github.com/codepr/orestes.git){:target="_blank"}

### JAS - Actor model with java RMI
-------------------------------------------------------------------
A system that abstract a simplified implementation of the actor model.
Originally started as a university project for a concurrent and distributed
programming course, I proceeded to add some features like support for remote
actors and a basic cluster system based on legacy RMI technology.

Unfinished project and likely bugged, it's not recommended to try it in a
real case of use.

[link](https://github.com/codepr/jas.git){:target="_blank"}
