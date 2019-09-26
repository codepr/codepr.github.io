---
layout: default
title: Projects
description: "Personal pet projects, mostly learning experiences. As a side note, these are just a part of all the code I produced in the last years, being it for a significant part experimental work."
permalink: /projects/
---

### Sol - Lightweight MQTT broker
----------------------------------------------------------------------
Oversimplified MQTT broker written from scratch, which mimick mosquitto
features. Implemented for learning how the protocol works, for now it supports
almost all MQTT v3.1.1 commands on linux platform; it relies on EPOLL interface
introduced on kernel 2.5.44.

[link](https://github.com/codepr/sol.git)

### Tasq - distributed task queue
---------------------------------------------------------------------
Very simple broker-less distributed Task queue that allow the scheduling of job
functions to be executed on local or remote workers. Can be seen as a Proof of
Concept leveraging ZMQ sockets and cloudpickle serialization capabilities as
well as a very basic actor system to handle different loads of work from
connecting clients.

[link](https://github.com/codepr/tasq.git)

### TrieDB - kv store based on a trie data structure
-------------------------------------------------------------------------
Single threaded Key-value store based on a Trie data structure. Trie is a kind
of trees in which each node is a prefix for a key, the node position define the
keys and the associated values are set on the last node of each key. They
provide a big-O runtime complexity of O(m) on worst case, for insertion and
lookup, where m is the length of the key. The main advantage is the possibility
to query the tree by prefix, executing range scans in an easy way.

Almost all commands supported has a "prefix" version which apply the command
itself on a prefix instead of a full key.

[link](https://github.com/codepr/triedb.git)

### Aiotunnel - HTTP(S) tunneling for local and remote port-forwarding
-------------------------------------------------------------------------------
Yet another HTTP tunnel, supports two modes; a direct one which open a local
port on the host machine and redirect all TCP data to the remote side of the
tunnel, which actually connect to the desired URL. A second one which require
the client part to be run on the target system we want to expose, the server
side on a (arguably) public machine (e.g. an AWS EC2) which expose a port to
communicate to our target system through HTTP.

[link](https://github.com/codepr/aiotunnel.git)

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

[link](https://github.com/codepr/orestes.git)

### JAS - Actor model with java RMI
-------------------------------------------------------------------
A system that abstract a simplified implementation of the actor model.
Originally started as a university project for a concurrent and distributed
programming course, I proceeded to add some features like support for remote
actors and a basic cluster system based on legacy RMI technology.

Unfinished project and likely bugged, it's not recommended to try it in a
real case of use.

[link](https://github.com/codepr/jas.git)
