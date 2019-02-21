---
layout: post
title: "Pet projects"
description: "Personal pet projects, mostly learning experiences. As a side note, these are just a part of all the code I produced in the last years, being it for a significant part experimental work."
---

<a href="https://github.com/codepr/tasq.git" target="_blank" class="pname">Tasq</a> - <span class="psub">A brokerless distributed task queue </span><span class="lang"> [Python] </span>
---------------------------------------------------------------------

Very simple broker-less distributed Task queue that allow the scheduling of job
functions to be executed on local or remote workers. Can be seen as a Proof of
Concept leveraging ZMQ sockets and cloudpickle serialization capabilities as
well as a very basic actor system to handle different loads of work from
connecting clients.

<a href="https://github.com/codepr/sizigy.git" target="_blank" class="pname">Sizigy</a> - <span class="psub"> A toy MQTT broker </span><span class="lang"> [C] </span>
-------------------------------------------------------------------------

Sizigy is a pet project born as a way to renew a bit of knowledge of the low
level programming in C. It's an MQTT message broker built upon epoll interface,
aiming to guarantee exactly once semantic. Distribution is still in development
as well as a lot of the core features, but it is already possible to play with
it. It's a good exercise to explore and better understand the TCP stack,
endianness, serialization and scalability.

<a href="https://github.com/codepr/vessel.git" target="_blank" class="pname">Vessel</a> - <span class="psub"> Epoll TCP server library supporting SSl </span> <span class="lang"> [C] </span>
-------------------------------------------------------------------------

Lightweight library to easily create and use a multithreaded epoll server. It
aims to make it simple to create pre-defined handler for the main
macro-operations that every server must fulfill: accepting new clients,
handling request and reply to clients.

<a href="https://github.com/codepr/aiotunnel.git" target="_blank" class="pname">Aiotunnel</a> - <span class="psub"> HTTPS Tunneling for local and remote port-forwarding </span> <span class="lang"> [Python] </span>
-------------------------------------------------------------------------------

Yet another HTTP tunnel, supports two modes; a direct one which open a local
port on the host machine and redirect all TCP data to the remote side of the
tunnel, which actually connect to the desired URL. A second one which require
the client part to be run on the target system we want to expose, the server
side on a (arguably) public machine (e.g. an AWS EC2) which expose a port to
communicate to our target system through HTTP.

<a href="https://github.com/codepr/memento.git" target="_blank" class="pname">Memento</a> - <span class="psub"> Remote hashmap over epoll tcp server </span> <span class="lang"> [C] </span>
---------------------------------------------------------------------------

Fairly simple hashmap implementation built on top of an epoll TCP server. A toy
project just for learning purpose, fascinated by the topic, i decided to try to
make a simpler version of redis/memcached to better understand how it works.
The project is a minimal Redis-like implementation with a text based protocol,
and like Redis, can be used as key-value in-memory store.

<a href="https://github.com/codepr/orestes.git" target="_blank" class="pname">Orestes</a> - <span class="psub"> Basic key-value store </span> <span class="lang"> [Haskell] </span>
--------------------------------------------------------------------------

Simple implementation of a distributed key-value server, aimed to learn basic
concepts of functional programming through Haskell. For the distribution it
uses cloud-haskell libraries, based on asynchronous message protocol like
Erlang distribution actor model. It works on both a cluster of machines or on a
single one according to a master-slave topology. It currently support just the
common operations `PUT`, `GET` and `DEL` and it lacks a suitable communication
protocol, on the other side it is perfectly usable by using a generic TCP
client like Telnet or Netcat issuing commands as strings.

<a href="https://github.com/codepr/jas.git" target="_blank" class="pname">Jas</a> - <span class="psub"> Actor model with java RMI </span> <span class="lang"> [Java] </span>
-------------------------------------------------------------------

A system that abstract a simplified implementation of the actor model.
Originally started as a university project for a concurrent and distributed
programming course, I proceeded to add some features like support for remote
actors and a basic cluster system based on legacy RMI technology.

Still under development, and likely bugged, it's not recommended to try it in a
real case of use.
