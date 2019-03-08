---
layout: post
title: "Sol - An MQTT broker from scratch. Part 5 - Handlers"
description: "Writing an MQTT broker from scratch, to really understand something you have to build it."
tags: [c, unix, tutorial]
---

This part will focus on the implementation of the handlers, they will be mapped
one-on-one with MQTT commands in an array, indexed by command type, making it
trivial to call the correct function depending on the packet type.

Before starting the main argument of the post, we add some missing parts to make
the previously written code usable.

Vim src/core.h:

**src/core.h**

{% highlight c %}

#ifndef CORE_H
#define CORE_H

#include "trie.h"
#include "list.h"
#include "hashtable.h"


struct topic {
    const char *name;
    List *subscribers;
};

/*
 * Main structure, a global instance will be instantiated at start, tracking
 * topics, connected clients and registered closures.
 */
struct sol {
    HashTable *clients;
    HashTable *closures;
    Trie topics;
};


struct session {
    List *subscriptions;
    // TODO add pending confirmed messages
};

/*
 * Wrapper structure around a connected client, each client can be a publisher
 * or a subscriber, it can be used to track sessions too.
 */
struct sol_client {
    char *client_id;
    int fd;
    struct session session;
};


struct subscriber {
    unsigned qos;
    struct sol_client *client;
};


struct topic *topic_create(const char *);

void topic_init(struct topic *, const char *);

void topic_add_subscriber(struct topic *, struct sol_client *, unsigned, bool);

void topic_del_subscriber(struct topic *, struct sol_client *, bool);

void sol_topic_put(struct sol *, struct topic *);

void sol_topic_del(struct sol *, const char *);

/* Find a topic by name and return it */
struct topic *sol_topic_get(struct sol *, const char *);


#endif

{% endhighlight %}

This module contains the major abstractions to handle the clients and his
interactions with the server, specifically:

- a client structure, represents the connected client
- a topic structure
- a subscriber structure
- a session structure, represents the session of the client, useful in case of
  `clean session` set to false for a client
- sol, the global structure handling the previous points
- a set of convenient helper functions

Here the implementation:

**src/core.c**

{% highlight c %}

#include <string.h>
#include "util.h"
#include "core.h"


static int compare_cid(void *c1, void *c2) {
    return strcmp(((struct subscriber *) c1)->client->client_id,
                  ((struct subscriber *) c2)->client->client_id);
}


struct topic *topic_create(const char *name) {
    struct topic *t = sol_malloc(sizeof(*t));
    topic_init(t, name);
    return t;
}


void topic_init(struct topic *t, const char *name) {
    t->name = name;
    t->subscribers = list_create(NULL);
}


void topic_add_subscriber(struct topic *t,
                          struct sol_client *client,
                          unsigned qos,
                          bool cleansession) {
    struct subscriber *sub = sol_malloc(sizeof(*sub));
    sub->client = client;
    sub->qos = qos;
    t->subscribers = list_push(t->subscribers, sub);

    // It must be added to the session if cleansession is false
    if (!cleansession)
        client->session.subscriptions =
            list_push(client->session.subscriptions, t);

}


void topic_del_subscriber(struct topic *t,
                          struct sol_client *client,
                          bool cleansession) {
    list_remove_node(t->subscribers, client, compare_cid);

    // TODO remomve in case of cleansession == false
}


void sol_topic_put(struct sol *sol, struct topic *t) {
    trie_insert(&sol->topics, t->name, t);
}


void sol_topic_del(struct sol *sol, const char *name) {
    trie_delete(&sol->topics, name);
}


struct topic *sol_topic_get(struct sol *sol, const char *name) {
    struct topic *ret_topic;
    trie_find(&sol->topics, name, (void *) &ret_topic);
    return ret_topic;
}

{% endhighlight %}


## Finally, the handlers

The first handler we'll add will be the `connect_handler`, which as the name
suggests will handle `CONNECT` packet coming just after connecting.

**src/server.c**

{% highlight c %}

static int connect_handler(struct closure *cb, union mqtt_packet *pkt) {

    // TODO just return error_code and handle it on `on_read`
    if (hashtable_exists(sol.clients,
                         (const char *) pkt->connect.payload.client_id)) {

        // Already connected client, 2 CONNECT packet should be interpreted as
        // a violation of the protocol, causing disconnection of the client

        sol_info("Received double CONNECT from %s, disconnecting client",
                 pkt->connect.payload.client_id);

        close(cb->fd);
        hashtable_del(sol.clients, (const char *) pkt->connect.payload.client_id);
        hashtable_del(sol.closures, cb->closure_id);

        // Update stats
        info.nclients--;
        info.nconnections--;

        return -REARM_W;
    }

    sol_info("New client connected as %s (c%i, k%u)",
             pkt->connect.payload.client_id,
             pkt->connect.bits.clean_session,
             pkt->connect.payload.keepalive);

    /*
     * Add the new connected client to the global map, if it is already
     * connected, kick him out accordingly to the MQTT v3.1.1 specs.
     */
    struct sol_client *new_client = sol_malloc(sizeof(*new_client));
    new_client->fd = cb->fd;
    const char *cid = (const char *) pkt->connect.payload.client_id;
    new_client->client_id = sol_strdup(cid);
    hashtable_put(sol.clients, cid, new_client);

    /* Substitute fd on callback with closure */
    cb->obj = new_client;

    /* Respond with a connack */
    union mqtt_packet *response = sol_malloc(sizeof(*response));
    unsigned char byte = CONNACK;

    // TODO check for session already present

    if (pkt->connect.bits.clean_session == false)
        new_client->session.subscriptions = list_create(NULL);

    unsigned char session_present = 0;
    unsigned char connect_flags = 0 | (session_present & 0x1) << 0;
    unsigned char rc = 0;  // 0 means connection accepted

    response->connack = *mqtt_packet_connack(byte, connect_flags, rc);

    cb->payload = bytestring_create(MQTT_ACK_LEN);
    unsigned char *p = pack_mqtt_packet(response, CONNACK_TYPE);
    memcpy(cb->payload->data, p, MQTT_ACK_LEN);
    sol_free(p);

    sol_debug("Sending CONNACK to %s (%u, %u)",
              pkt->connect.payload.client_id,
              session_present, rc);

    sol_free(response);

    return REARM_W;
}

{% endhighlight %}

Essentially it behave exactly as defined by the protocol standard, except for
the *clean session* thing, which for now we ignore; if a double `CONNECT`
packet is received it kick out the connected clients that sent it, otherwise it
schedule a response to it with a `CONNACK`, that will be handled by the
`on_write` handler.

Next command, `DISCONNECT`:

**src/server.c**

{% highlight c %}

static int disconnect_handler(struct closure *cb, union mqtt_packet *pkt) {

    // TODO just return error_code and handle it on `on_read`

    /* Handle disconnection request from client */
    struct sol_client *c = cb->obj;

    sol_debug("Received DISCONNECT from %s", c->client_id);

    close(c->fd);
    hashtable_del(sol.clients, c->client_id);
    hashtable_del(sol.closures, cb->closure_id);

    // Update stats
    info.nclients--;
    info.nconnections--;

    // TODO remove from all topic where it subscribed
    return -REARM_W;
}

{% endhighlight %}

Straight forward, just log the disconnection, update the infos, close the fd,
remove the client from the global map and return a negative code, neat.

Let's move to a more interesting operation, `SUBSCRIBE`, this is where our
trie structure kick-in, in fact we have to:

- Iterate through the list of tuples (topic, QoS) and for each
    - If the topic does not exists we create it
    - Add the client to the subscribers lists of the given topic
    - If the topic ends with "#" we have to subscribe to the given
      topic and all of his children, this can be done recursively
      in a trivial manner thanks to the nature of the trie struct
    - Add to session in case of `clean_session` false, still need
      some implementation here through
- Answer with a `SUBACK`

Nothing special for the `UNSUBSCRIBE` command, remove the client from the
specified topic and answer with an `UNSUBACK`:

**src/server.c**

{% highlight c %}

/* Recursive auxiliary function to subscribe to all children of a given topic */
static void recursive_subscription(struct trie_node *node, void *arg) {

    if (!node || !node->data)
        return;

    struct list_node *child = node->children->head;
    for (; child; child = child->next)
        recursive_subscription(child->data, arg);

    struct topic *t = node->data;

    struct subscriber *s = arg;

    t->subscribers = list_push(t->subscribers, s);
}


static int subscribe_handler(struct closure *cb, union mqtt_packet *pkt) {

    struct sol_client *c = cb->obj;
    bool wildcard = false;
    bool alloced = false;

    /*
     * We respond to the subscription request with SUBACK and a list of QoS in
     * the same exact order of reception
     */
    unsigned char rcs[pkt->subscribe.tuples_len];

    /* Subscribe packets contains a list of topics and QoS tuples */
    for (unsigned i = 0; i < pkt->subscribe.tuples_len; i++) {

        sol_debug("Received SUBSCRIBE from %s", c->client_id);

        /*
         * Check if the topic exists already or in case create it and store in
         * the global map
         */
        char *topic = (char *) pkt->subscribe.tuples[i].topic;

        sol_debug("\t%s (QoS %i)", topic, pkt->subscribe.tuples[i].qos);

        /* Recursive subscribe to all children topics if the topic ends with "/#" */
        if (topic[pkt->subscribe.tuples[i].topic_len - 1] == '#' &&
            topic[pkt->subscribe.tuples[i].topic_len - 2] == '/') {
            topic = remove_occur(topic, '#');
            wildcard = true;
        } else if (topic[pkt->subscribe.tuples[i].topic_len - 1] != '/') {
            topic = append_string((char *) pkt->subscribe.tuples[i].topic, "/", 1);
            alloced = true;
        }

        struct topic *t = sol_topic_get(&sol, topic);

        // TODO check for callback correctly set to obj

        if (!t) {
            t = topic_create(sol_strdup(topic));
            sol_topic_put(&sol, t);
        } else if (wildcard == true) {
            struct subscriber *sub = sol_malloc(sizeof(*sub));
            sub->client = cb->obj;
            sub->qos = pkt->subscribe.tuples[i].qos;
            trie_prefix_map_tuple(&sol.topics, topic,
                                  recursive_subscription, sub);
        }

        // Clean session true for now
        topic_add_subscriber(t, cb->obj, pkt->subscribe.tuples[i].qos, true);

        if (alloced)
            sol_free(topic);

        rcs[i] = pkt->subscribe.tuples[i].qos;
    }

    struct mqtt_suback *suback = mqtt_packet_suback(SUBACK,
                                                    pkt->subscribe.pkt_id,
                                                    rcs,
                                                    pkt->subscribe.tuples_len);

    mqtt_packet_release(pkt, SUBSCRIBE_TYPE);
    pkt->suback = *suback;
    unsigned char *packed = pack_mqtt_packet(pkt, SUBACK_TYPE);
    size_t len = MQTT_HEADER_LEN + sizeof(uint16_t) + pkt->subscribe.tuples_len;
    cb->payload = bytestring_create(len);
    memcpy(cb->payload->data, packed, len);
    sol_free(packed);

    mqtt_packet_release(pkt, SUBACK_TYPE);
    sol_free(suback);

    sol_debug("Sending SUBACK to %s", c->client_id);

    return REARM_W;
}


static int unsubscribe_handler(struct closure *cb, union mqtt_packet *pkt) {

    struct sol_client *c = cb->obj;

    sol_debug("Received UNSUBSCRIBE from %s", c->client_id);

    pkt->ack = *mqtt_packet_ack(UNSUBACK, pkt->unsubscribe.pkt_id);

    unsigned char *packed = pack_mqtt_packet(pkt, UNSUBACK_TYPE);
    cb->payload = bytestring_create(MQTT_ACK_LEN);
    memcpy(cb->payload->data, packed, MQTT_ACK_LEN);
    sol_free(packed);

    sol_debug("Sending UNSUBACK to %s", c->client_id);

    return REARM_W;
}


{% endhighlight %}

The `PUBLISH` handler is the longer of our handlers, but it's fairly easy to
follow

- create the topic if it does not exists
- based upon the QoS of the message, it schedules the correct `ACK` (`PUBACK` for
  an **at least once** level, `PUBREC` for an **exactly once** level, nothing for
  the `at most once` level)
- Forward the publish packed with the updated QoS to all subscribers of the topic,
  the QoS have to be updated to the QoS of the subscriber, which is the maximum
  QoS level that can be received by the subscriber.

<br>

**Publish QoS 2 message on a topic with a single subscriber on QoS 1**

<br>

<center>
{% include image.html path="QoS2-sample.png" path-detail="QoS2-sample.png" alt="QoS2 sequential diagram" %}
</center>
<br>

**src/server.c**

{% highlight c %}

static int publish_handler(struct closure *cb, union mqtt_packet *pkt) {

    struct sol_client *c = cb->obj;

    sol_debug("Received PUBLISH from %s (d%i, q%u, r%i, m%u, %s, ... (%i bytes))",
              c->client_id,
              pkt->publish.header.bits.dup,
              pkt->publish.header.bits.qos,
              pkt->publish.header.bits.retain,
              pkt->publish.pkt_id,
              pkt->publish.topic,
              pkt->publish.payloadlen);

    info.messages_recv++;

    char *topic = (char *) pkt->publish.topic;
    bool alloced = false;
    unsigned char qos = pkt->publish.header.bits.qos;

    /*
     * For convenience we assure that all topics ends with a '/', indicating a
     * hierarchical level
     */
    if (topic[pkt->publish.topiclen - 1] != '/') {
        topic = append_string((char *) pkt->publish.topic, "/", 1);
        alloced = true;
    }

    /*
     * Retrieve the topic from the global map, if it wasn't created before,
     * create a new one with the name selected
     */
    struct topic *t = sol_topic_get(&sol, topic);

    if (!t) {
        t = topic_create(sol_strdup(topic));
        sol_topic_put(&sol, t);
    }

    // Not the best way to handle this
    if (alloced == true)
        sol_free(topic);

    size_t publen;
    unsigned char *pub;

    struct list_node *cur = t->subscribers->head;
    for (; cur; cur = cur->next) {

        publen = MQTT_HEADER_LEN + sizeof(uint16_t) +
            pkt->publish.topiclen + pkt->publish.payloadlen;

        struct subscriber *sub = cur->data;
        struct sol_client *sc = sub->client;

        /* Update QoS according to subscriber's one */
        pkt->publish.header.bits.qos = sub->qos;

        if (pkt->publish.header.bits.qos > AT_MOST_ONCE)
            publen += sizeof(uint16_t);

        pub = pack_mqtt_packet(pkt, PUBLISH_TYPE);

        ssize_t sent;
        if ((sent = send_bytes(sc->fd, pub, publen)) < 0)
            sol_error("Error publishing to %s: %s",
                      sc->client_id, strerror(errno));

        // Update information stats
        info.bytes_sent += sent;

        sol_debug("Sending PUBLISH to %s (d%i, q%u, r%i, m%u, %s, ... (%i bytes))",
                  sc->client_id,
                  pkt->publish.header.bits.dup,
                  pkt->publish.header.bits.qos,
                  pkt->publish.header.bits.retain,
                  pkt->publish.pkt_id,
                  pkt->publish.topic,
                  pkt->publish.payloadlen);

        info.messages_sent++;

        sol_free(pub);
    }

    // TODO free publish

    if (qos == AT_LEAST_ONCE) {

        mqtt_puback *puback = mqtt_packet_ack(PUBACK, pkt->publish.pkt_id);

        mqtt_packet_release(pkt, PUBLISH_TYPE);

        pkt->ack = *puback;

        unsigned char *packed = pack_mqtt_packet(pkt, PUBACK_TYPE);
        cb->payload = bytestring_create(MQTT_ACK_LEN);
        memcpy(cb->payload->data, packed, MQTT_ACK_LEN);
        sol_free(packed);

        sol_debug("Sending PUBACK to %s", c->client_id);

        return REARM_W;

    } else if (qos == EXACTLY_ONCE) {

        // TODO add to a hashtable to track PUBREC clients last

        mqtt_pubrec *pubrec = mqtt_packet_ack(PUBREC, pkt->publish.pkt_id);

        mqtt_packet_release(pkt, PUBLISH_TYPE);

        pkt->ack = *pubrec;

        unsigned char *packed = pack_mqtt_packet(pkt, PUBREC_TYPE);
        cb->payload = bytestring_create(MQTT_ACK_LEN);
        memcpy(cb->payload->data, packed, MQTT_ACK_LEN);
        sol_free(packed);

        sol_debug("Sending PUBREC to %s", c->client_id);

        return REARM_W;

    }

    mqtt_packet_release(pkt, PUBLISH_TYPE);

    /*
     * We're in the case of AT_MOST_ONCE QoS level, we don't need to sent out
     * any byte, it's a fire-and-forget.
     */
    return REARM_R;
}

{% endhighlight %}

All the remaining `ACK` handlers now, they basically all the same, for now we
limit ourselves to just log their execution, in the future we'll handle the
message based on the QoS deliverance.

The last one, is the `PINGREQ` handler, it's only purpose is to guarantee the
health of connected clients who are inactive for some time, receiving one
expects a `PINGRESP` as answer.

**src/server.c**

{% highlight c %}

static int puback_handler(struct closure *cb, union mqtt_packet *pkt) {

    sol_debug("Received PUBACK from %s",
              ((struct sol_client *) cb->obj)->client_id);

    // TODO Remove from pending PUBACK clients map

    return REARM_R;
}


static int pubrec_handler(struct closure *cb, union mqtt_packet *pkt) {

    struct sol_client *c = cb->obj;

    sol_debug("Received PUBREC from %s", c->client_id);

    mqtt_pubrel *pubrel = mqtt_packet_ack(PUBREL, pkt->publish.pkt_id);

    pkt->ack = *pubrel;

    unsigned char *packed = pack_mqtt_packet(pkt, PUBREC_TYPE);
    cb->payload = bytestring_create(MQTT_ACK_LEN);
    memcpy(cb->payload->data, packed, MQTT_ACK_LEN);
    sol_free(packed);

    sol_debug("Sending PUBREL to %s", c->client_id);

    return REARM_W;
}


static int pubrel_handler(struct closure *cb, union mqtt_packet *pkt) {

    sol_debug("Received PUBREL from %s",
              ((struct sol_client *) cb->obj)->client_id);

    mqtt_pubcomp *pubcomp = mqtt_packet_ack(PUBCOMP, pkt->publish.pkt_id);

    pkt->ack = *pubcomp;

    unsigned char *packed = pack_mqtt_packet(pkt, PUBCOMP_TYPE);
    cb->payload = bytestring_create(MQTT_ACK_LEN);
    memcpy(cb->payload->data, packed, MQTT_ACK_LEN);
    sol_free(packed);

    sol_debug("Sending PUBCOMP to %s",
              ((struct sol_client *) cb->obj)->client_id);

    return REARM_W;
}


static int pubcomp_handler(struct closure *cb, union mqtt_packet *pkt) {

    sol_debug("Received PUBCOMP from %s",
              ((struct sol_client *) cb->obj)->client_id);

    // TODO Remove from pending PUBACK clients map

    return REARM_R;
}


static int pingreq_handler(struct closure *cb, union mqtt_packet *pkt) {

    sol_debug("Received PINGREQ from %s",
              ((struct sol_client *) cb->obj)->client_id);

    pkt->header = *mqtt_packet_header(PINGRESP);
    unsigned char *packed = pack_mqtt_packet(pkt, PINGRESP_TYPE);
    cb->payload = bytestring_create(MQTT_HEADER_LEN);
    memcpy(cb->payload->data, packed, MQTT_HEADER_LEN);
    sol_free(packed);

    sol_debug("Sending PINGRESP to %s",
              ((struct sol_client *) cb->obj)->client_id);

    return REARM_W;
}

{% endhighlight %}

Our lightweight broker is taking shape, code should now be enough to try and
play a bit, using `mosquitto_sub` and `mosquitto_pub` or Python `paho-mqtt`
library.

Let's add the last brick, a main:

**src/sol.c**

{% highlight c %}

#define _POSIX_C_SOURCE 2
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include "util.h"
#include "server.h"


int main (int argc, char **argv) {

    char *addr = DEFAULT_HOSTNAME;
    char *port = DEFAULT_PORT;
    int debug = 0;
    int opt;

    while ((opt = getopt(argc, argv, "a:c:p:m:vn:")) != -1) {
        switch (opt) {
            case 'a':
                addr = optarg;
                break;
            case 'c':
                confpath = optarg;
                break;
            case 'p':
                port = optarg;
                break;
            case 'v':
                debug = 1;
                break;
            default:
                fprintf(stderr,
                        "Usage: %s [-a addr] [-p port] [-c conf] [-v]\n",
                        argv[0]);
                exit(EXIT_FAILURE);
        }
    }

    start_server(addr, port);

    return 0;
}

{% endhighlight %}

Short and clean, for compilation I'd like to write custom `Makefile` usually,
but this time, as shown on the folder tree, I'll consider using a
`CMakeLists.txt` defined template ad `cmake` to generate it:

{% highlight bash %}

cmake_minimum_required(VERSION 2.8)

project(sol)

OPTION(DEBUG "add debug flags" OFF)

if (DEBUG)
    message(STATUS "Configuring build for debug")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wunused -Werror -std=c11 -O3 -pedantic -luuid -ggdb -fsanitize=address -fsanitize=undefined -fno-omit-frame-pointer -pg")
else (DEBUG)
    message(STATUS "Configuring build for production")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wunused -Werror -Wextra -std=c11 -O3 -pedantic -luuid")
endif (DEBUG)

set(EXECUTABLE_OUTPUT_PATH ${CMAKE_SOURCE_DIR})

file(GLOB SOURCES src/*.c)

set(AUTHOR "Andrea Giacomo Baldan")
set(LICENSE "BSD2 license")

# Executable
add_executable(sol ${SOURCES})

{% endhighlight %}

The only part worth a note is the `DEBUG` flag that I added, it makes cmake
generate a different `Makefile` that compile the sources with some additional
flags to catch and signal memory leaks and undefined behaviours.

So the next move is to generate the `Makefile`

{% highlight bash %}

$ cmake -DDEBUG=1 .

{% endhighlight %}

and compile the sources:

{% highlight bash %}

$ make

{% endhighlight %}
