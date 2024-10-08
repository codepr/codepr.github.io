---
layout: post
title: "Sol - An MQTT broker from scratch. Part 6 - Handlers"
description: "Writing an MQTT broker from scratch, to really understand something you have to build it."
categories: c unix tutorial
---

This part will focus on the implementation of the handlers, they will be mapped
one-on-one with MQTT commands in an array, indexed by command type, making it
trivial to call the correct function depending on the packet type.

<!--more-->

Before starting the main argument of the post, we add some missing parts to make
the previously written code usable.

Let's fire vim src/core.h:

<hr>
**src/core.h**
<hr>

```c

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

```
<hr>

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

<hr>
**src/core.c**
<hr>

```c

#include <string.h>
#include <stdlib.h>
#include "core.h"

static int compare_cid(void *c1, void *c2) {
    return strcmp(((struct subscriber *) c1)->client->client_id,
                  ((struct subscriber *) c2)->client->client_id);
}

struct topic *topic_create(const char *name) {
    struct topic *t = malloc(sizeof(*t));
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
    struct subscriber *sub = malloc(sizeof(*sub));
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

```
<hr>

### Finally, the handlers

Handlers are the functions that will be called on the `on_read` callback, as the
name suggests, they handle commands, after being done, they optionally set a payload
ready for being sent to the client and return a code, which can be:

- REARM_W, imply the rearming of the descriptor setting the next callback to `on_write`
  there's data to send to the client
- REARM_R, in this case there's nothing to tell to the client, the file descriptor will
  be rearmed with the callback set to `on_read` again
- -REARM_W this particular case has no code defined, but use the REARM_W and negate it,
  it means that the client disconnected.

The first handler we'll add will be the `connect_handler`, which as the name
suggests will handle CONNECT packet coming just after a TCP connection happens.

<hr>
**src/server.c**
<hr>

```c

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
    struct sol_client *new_client = malloc(sizeof(*new_client));
    new_client->fd = cb->fd;
    const char *cid = (const char *) pkt->connect.payload.client_id;
    new_client->client_id = strdup(cid);
    hashtable_put(sol.clients, cid, new_client);

    /* Substitute fd on callback with closure */
    cb->obj = new_client;

    /* Respond with a connack */
    union mqtt_packet *response = malloc(sizeof(*response));
    unsigned char byte = CONNACK_BYTE;

    // TODO check for session already present

    if (pkt->connect.bits.clean_session == false)
        new_client->session.subscriptions = list_create(NULL);

    unsigned char session_present = 0;
    unsigned char connect_flags = 0 | (session_present & 0x1) << 0;
    unsigned char rc = 0;  // 0 means connection accepted

    response->connack = *mqtt_packet_connack(byte, connect_flags, rc);

    cb->payload = bytestring_create(MQTT_ACK_LEN);
    unsigned char *p = pack_mqtt_packet(response, CONNACK);
    memcpy(cb->payload->data, p, MQTT_ACK_LEN);
    free(p);

    sol_debug("Sending CONNACK to %s (%u, %u)",
              pkt->connect.payload.client_id,
              session_present, rc);

    free(response);

    return REARM_W;
}

```
<hr>

Essentially it behaves exactly as defined by the protocol standard, except for
the *clean session* thing, which for now we ignore; if a double CONNECT
packet is received it kicks out the connected clients that sent the request,
otherwise it schedules a response with a CONNACK, that will be handled by
the `on_write` handler.

Next command, DISCONNECT:

<hr>
**src/server.c**
<hr>

```c

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

```
<hr>

Straight forward, just log the disconnection, update the info, close the fd,
remove the client from the global map and return a negative code, neat.

Let's move to a more interesting operation, SUBSCRIBE, this is where our
trie structure kick-in, in fact we have to:

- Iterate through the list of tuples (topic, QoS) and for each
    - If the topic does not exist we create it
    - Add the client to the subscribers lists of the given topic
    - If the topic ends with "#" we have to subscribe to the given
      topic and all of his children, this can be done recursively
      in a trivial manner thanks to the nature of the trie struct
    - Add to session in case of `clean_session` false, still need
      some implementation here through
- Answer with a SUBACK

Nothing special for the UNSUBSCRIBE command, remove the client from the
specified topic and answer with an UNSUBACK:

<hr>
**src/server.c**
<hr>

```c

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
            t = topic_create(strdup(topic));
            sol_topic_put(&sol, t);
        } else if (wildcard == true) {
            struct subscriber *sub = malloc(sizeof(*sub));
            sub->client = cb->obj;
            sub->qos = pkt->subscribe.tuples[i].qos;
            trie_prefix_map_tuple(&sol.topics, topic,
                                  recursive_subscription, sub);
        }

        // Clean session true for now
        topic_add_subscriber(t, cb->obj, pkt->subscribe.tuples[i].qos, true);
        if (alloced)
            free(topic);
        rcs[i] = pkt->subscribe.tuples[i].qos;
    }
    struct mqtt_suback *suback = mqtt_packet_suback(SUBACK_BYTE,
                                                    pkt->subscribe.pkt_id,
                                                    rcs,
                                                    pkt->subscribe.tuples_len);
    mqtt_packet_release(pkt, SUBSCRIBE);
    pkt->suback = *suback;
    unsigned char *packed = pack_mqtt_packet(pkt, SUBACK);
    size_t len = MQTT_HEADER_LEN + sizeof(uint16_t) + pkt->subscribe.tuples_len;
    cb->payload = bytestring_create(len);
    memcpy(cb->payload->data, packed, len);
    free(packed);
    mqtt_packet_release(pkt, SUBACK);
    free(suback);
    sol_debug("Sending SUBACK to %s", c->client_id);
    return REARM_W;
}

static int unsubscribe_handler(struct closure *cb, union mqtt_packet *pkt) {
    struct sol_client *c = cb->obj;
    sol_debug("Received UNSUBSCRIBE from %s", c->client_id);
    pkt->ack = *mqtt_packet_ack(UNSUBACK_BYTE, pkt->unsubscribe.pkt_id);
    unsigned char *packed = pack_mqtt_packet(pkt, UNSUBACK);
    cb->payload = bytestring_create(MQTT_ACK_LEN);
    memcpy(cb->payload->data, packed, MQTT_ACK_LEN);
    free(packed);
    sol_debug("Sending UNSUBACK to %s", c->client_id);
    return REARM_W;
}

```
<hr>

The PUBLISH handler is the longer of our handlers, but it's fairly easy to
follow

- create the topic if it does not exist
- based upon the QoS of the message, it schedules the correct ACK (PUBACK for
  an **at least once** level, PUBREC for an **exactly once** level, nothing for
  the **at most once** level)
- Forward the publish packed with the updated QoS to all subscribers of the topic,
  the QoS have to be updated to the QoS of the subscriber, which is the maximum
  QoS level that can be received by the subscriber.

<br>

**Publish QoS 2 message on a topic with a single subscriber on QoS 1**

<br>
![QoS2 sequential diagram]({{site.url}}{{site.baseurl}}/assets/images/QoS2-sample.png#content-image)
<br>

<hr>
**src/server.c**
<hr>

```c

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
        t = topic_create(strdup(topic));
        sol_topic_put(&sol, t);
    }

    // Not the best way to handle this
    if (alloced == true)
        free(topic);
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
        int remaininglen_offset = 0;
        if ((publen - 1) > 0x200000)
            remaininglen_offset = 3;
        else if ((publen - 1) > 0x4000)
            remaininglen_offset = 2;
        else if ((publen - 1) > 0x80)
            remaininglen_offset = 1;
        publen += remaininglen_offset;
        pub = pack_mqtt_packet(pkt, PUBLISH);
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
        free(pub);
    }

    // TODO free publish
    if (qos == AT_LEAST_ONCE) {
        mqtt_puback *puback = mqtt_packet_ack(PUBACK_BYTE, pkt->publish.pkt_id);
        mqtt_packet_release(pkt, PUBLISH);
        pkt->ack = *puback;
        unsigned char *packed = pack_mqtt_packet(pkt, PUBACK);
        cb->payload = bytestring_create(MQTT_ACK_LEN);
        memcpy(cb->payload->data, packed, MQTT_ACK_LEN);
        free(packed);
        sol_debug("Sending PUBACK to %s", c->client_id);
        return REARM_W;
    } else if (qos == EXACTLY_ONCE) {
        // TODO add to a hashtable to track PUBREC clients last
        mqtt_pubrec *pubrec = mqtt_packet_ack(PUBREC_BYTE, pkt->publish.pkt_id);
        mqtt_packet_release(pkt, PUBLISH);
        pkt->ack = *pubrec;
        unsigned char *packed = pack_mqtt_packet(pkt, PUBREC);
        cb->payload = bytestring_create(MQTT_ACK_LEN);
        memcpy(cb->payload->data, packed, MQTT_ACK_LEN);
        free(packed);
        sol_debug("Sending PUBREC to %s", c->client_id);
        return REARM_W;
    }
    mqtt_packet_release(pkt, PUBLISH);
    /*
     * We're in the case of AT_MOST_ONCE QoS level, we don't need to sent out
     * any byte, it's a fire-and-forget.
     */
    return REARM_R;
}

```
<hr>

All the remaining ACK handlers now, they're basically all the same, for now we
limit ourselves to just log their execution, in the future we'll handle the
message based on the QoS deliverance.

The last one, is the PINGREQ handler, it's only purpose is to guarantee the
health of connected clients who are inactive for some time, receiving one
expects a PINGRESP as answer.

<hr>
**src/server.c**
<hr>

```c

static int puback_handler(struct closure *cb, union mqtt_packet *pkt) {
    sol_debug("Received PUBACK from %s",
              ((struct sol_client *) cb->obj)->client_id);
    // TODO Remove from pending PUBACK clients map
    return REARM_R;
}

static int pubrec_handler(struct closure *cb, union mqtt_packet *pkt) {
    struct sol_client *c = cb->obj;
    sol_debug("Received PUBREC from %s", c->client_id);
    mqtt_pubrel *pubrel = mqtt_packet_ack(PUBREL_BYTE, pkt->publish.pkt_id);
    pkt->ack = *pubrel;
    unsigned char *packed = pack_mqtt_packet(pkt, PUBREC);
    cb->payload = bytestring_create(MQTT_ACK_LEN);
    memcpy(cb->payload->data, packed, MQTT_ACK_LEN);
    free(packed);
    sol_debug("Sending PUBREL to %s", c->client_id);
    return REARM_W;
}

static int pubrel_handler(struct closure *cb, union mqtt_packet *pkt) {
    sol_debug("Received PUBREL from %s",
              ((struct sol_client *) cb->obj)->client_id);
    mqtt_pubcomp *pubcomp = mqtt_packet_ack(PUBCOMP_BYTE, pkt->publish.pkt_id);
    pkt->ack = *pubcomp;
    unsigned char *packed = pack_mqtt_packet(pkt, PUBCOMP);
    cb->payload = bytestring_create(MQTT_ACK_LEN);
    memcpy(cb->payload->data, packed, MQTT_ACK_LEN);
    free(packed);
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
    pkt->header = *mqtt_packet_header(PINGRESP_BYTE);
    unsigned char *packed = pack_mqtt_packet(pkt, PINGRESP);
    cb->payload = bytestring_create(MQTT_HEADER_LEN);
    memcpy(cb->payload->data, packed, MQTT_HEADER_LEN);
    free(packed);
    sol_debug("Sending PINGRESP to %s",
              ((struct sol_client *) cb->obj)->client_id);
    return REARM_W;
}

```
<hr>

Our lightweight broker is taking shape, code should now be enough to try and
play a bit, using `mosquitto_sub` and `mosquitto_pub` or Python `paho-mqtt`
library.

We just need a configuration module to have better control on the general
settings of the system and after that we will soon to be finished.
The format of the configuration file will be the classical key value one
used by most services on Linux, something like this:

<hr>
**conf/sol.conf**
<hr>

```bash

# Sol configuration file, uncomment and edit desired configuration

# Network configuration

# Uncomment ip_address and ip_port to set socket family to TCP, if unix_socket
# is set, UNIX family socket will be used

# ip_address 127.0.0.1
# ip_port 9090

unix_socket /tmp/sol.sock

# Logging configuration

# Could be either DEBUG, INFO/INFORMATION, WARNING, ERROR
log_level DEBUG

log_path /tmp/sol.log

# Max memory to be used, after which the system starts to reclaim memory by
# freeing older items stored
max_memory 2GB

# Max memory that will be allocated for each request
max_request_size 50MB

# TCP backlog, size of the complete connection queue
tcp_backlog 128

# Interval of time between one stats publish on $SOL topics and the subsequent
stats_publish_interval 10s

```
<hr>

So, let's define a new header:

<hr>
**src/config.h**
<hr>

```c

#ifndef CONFIG_H
#define CONFIG_H

#include <stdio.h>

// Default parameters
#define VERSION                     "0.0.1"
#define DEFAULT_SOCKET_FAMILY       INET
#define DEFAULT_LOG_LEVEL           DEBUG
#define DEFAULT_LOG_PATH            "/tmp/sol.log"
#define DEFAULT_CONF_PATH           "/etc/sol/sol.conf"
#define DEFAULT_HOSTNAME            "127.0.0.1"
#define DEFAULT_PORT                "1883"
#define DEFAULT_MAX_MEMORY          "2GB"
#define DEFAULT_MAX_REQUEST_SIZE    "2MB"
#define DEFAULT_STATS_INTERVAL      "10s"

struct config {
    /* Sol version <MAJOR.MINOR.PATCH> */
    const char *version;
    /* Eventfd to break the epoll_wait loop in case of signals */
    int run;
    /* Logging level, to be set by reading configuration */
    int loglevel;
    /* Epoll wait timeout, define even the number of times per second that the
       system will check for expired keys */
    int epoll_timeout;
    /* Socket family (Unix domain or TCP) */
    int socket_family;
    /* Log file path */
    char logpath[0xFF];
    /* Hostname to listen on */
    char hostname[0xFF];
    /* Port to open while listening, only if socket_family is INET,
     * otherwise it's ignored */
    char port[0xFF];
    /* Max memory to be used, after which the system starts to reclaim back by
     * freeing older items stored */
    size_t max_memory;
    /* Max memory request can allocate */
    size_t max_request_size;
    /* TCP backlog size */
    int tcp_backlog;
    /* Delay between every automatic publish of broker stats on topic */
    size_t stats_pub_interval;
};

extern struct config *conf;

void config_set_default(void);
void config_print(void);
int config_load(const char *);

char *time_to_string(size_t);
char *memory_to_string(size_t);

#endif

```
<hr>

The configuration explains itself, for now we want control over host and port
to listen on, the socket family (between TCP and UNIX) log level and log file
path, plus some minor utilities regarding network communication tuning.

The implementation will involve mainly utility functions to parse strings
and to read from file the configuration and populate the global configuration
structure:

<hr>
**src/config.c**
<hr>

```c

#include <ctype.h>
#include <string.h>
#include <stdlib.h>
#include <assert.h>
#include <sys/socket.h>
#include <sys/eventfd.h>
#include "util.h"
#include "config.h"
#include "network.h"

/* The main configuration structure */
static struct config config;
struct config *conf;

struct llevel {
    const char *lname;
    int loglevel;
};

static const struct llevel lmap[5] = {
    {"DEBUG", DEBUG},
    {"WARNING", WARNING},
    {"ERROR", ERROR},
    {"INFO", INFORMATION},
    {"INFORMATION", INFORMATION}
};

static size_t read_memory_with_mul(const char *memory_string) {

    /* Extract digit part */
    size_t num = parse_int(memory_string);
    int mul = 1;

    /* Move the pointer forward till the first non-digit char */
    while (isdigit(*memory_string)) memory_string++;

    /* Set multiplier */
    if (STREQ(memory_string, "kb", 2))
        mul = 1024;
    else if (STREQ(memory_string, "mb", 2))
        mul = 1024 * 1024;
    else if (STREQ(memory_string, "gb", 2))
        mul = 1024 * 1024 * 1024;

    return num * mul;
}

static size_t read_time_with_mul(const char *time_string) {

    /* Extract digit part */
    size_t num = parse_int(time_string);
    int mul = 1;

    /* Move the pointer forward till the first non-digit char */
    while (isdigit(*time_string)) time_string++;

    /* Set multiplier */
    switch (*time_string) {
        case 'm':
            mul = 60;
            break;
        case 'd':
            mul = 60 * 60 * 24;
            break;
        default:
            mul = 1;
            break;
    }

    return num * mul;
}

/* Format a memory in bytes to a more human-readable form, e.g. 64b or 18Kb
 * instead of huge numbers like 130230234 bytes */
char *memory_to_string(size_t memory) {
    int numlen = 0;
    int translated_memory = 0;
    char *mstring = NULL;
    if (memory < 1024) {
        translated_memory = memory;
        numlen = number_len(translated_memory);
        // +1 for 'b' +1 for nul terminating
        mstring = malloc(numlen + 1);
        snprintf(mstring, numlen + 1, "%db", translated_memory);
    } else if (memory < 1048576) {
        translated_memory = memory / 1024;
        numlen = number_len(translated_memory);
        // +2 for 'Kb' +1 for nul terminating
        mstring = malloc(numlen + 2);
        snprintf(mstring, numlen + 2, "%dKb", translated_memory);
    } else if (memory < 1073741824) {
        translated_memory = memory / (1024 * 1024);
        numlen = number_len(translated_memory);
        // +2 for 'Mb' +1 for nul terminating
        mstring = malloc(numlen + 2);
        snprintf(mstring, numlen + 2, "%dMb", translated_memory);
    } else {
        translated_memory = memory / (1024 * 1024 * 1024);
        numlen = number_len(translated_memory);
        // +2 for 'Gb' +1 for nul terminating
        mstring = malloc(numlen + 2);
        snprintf(mstring, numlen + 2, "%dGb", translated_memory);
    }
    return mstring;
}

/* Purely utility function, format a time in seconds to a more human-readable
 * form, e.g. 2m or 4h instead of huge numbers */
char *time_to_string(size_t time) {
    int numlen = 0;
    int translated_time = 0;
    char *tstring = NULL;
    if (time < 60) {
        translated_time = time;
        numlen = number_len(translated_time);
        // +1 for 's' +1 for nul terminating
        tstring = malloc(numlen + 1);
        snprintf(tstring, numlen + 1, "%ds", translated_time);
    } else if (time < 60 * 60) {
        translated_time = time / 60;
        numlen = number_len(translated_time);
        // +1 for 'm' +1 for nul terminating
        tstring = malloc(numlen + 1);
        snprintf(tstring, numlen + 1, "%dm", translated_time);
    } else if (time < 60 * 60 * 24) {
        translated_time = time / (60 * 60);
        numlen = number_len(translated_time);
        // +1 for 'h' +1 for nul terminating
        tstring = malloc(numlen + 1);
        snprintf(tstring, numlen + 1, "%dh", translated_time);
    } else {
        translated_time = time / (60 * 60 * 24);
        numlen = number_len(translated_time);
        // +1 for 'd' +1 for nul terminating
        tstring = malloc(numlen + 1);
        snprintf(tstring, numlen + 1, "%dd", translated_time);
    }
    return tstring;
}

/* Set configuration values based on what is read from the persistent
   configuration on disk */
static void add_config_value(const char *key, const char *value) {
    size_t klen = strlen(key);
    size_t vlen = strlen(value);
    if (STREQ("log_level", key, klen) == true) {
        for (int i = 0; i < 3; i++) {
            if (STREQ(lmap[i].lname, value, vlen) == true)
                config.loglevel = lmap[i].loglevel;
        }
    } else if (STREQ("log_path", key, klen) == true) {
        strcpy(config.logpath, value);
    } else if (STREQ("unix_socket", key, klen) == true) {
        config.socket_family = UNIX;
        strcpy(config.hostname, value);
    } else if (STREQ("ip_address", key, klen) == true) {
        config.socket_family = INET;
        strcpy(config.hostname, value);
    } else if (STREQ("ip_port", key, klen) == true) {
        strcpy(config.port, value);
    } else if (STREQ("max_memory", key, klen) == true) {
        config.max_memory = read_memory_with_mul(value);
    } else if (STREQ("max_request_size", key, klen) == true) {
        config.max_request_size = read_memory_with_mul(value);
    } else if (STREQ("tcp_backlog", key, klen) == true) {
        int tcp_backlog = parse_int(value);
        config.tcp_backlog = tcp_backlog <= SOMAXCONN ? tcp_backlog : SOMAXCONN;
    } else if (STREQ("stats_publish_interval", key, klen) == true) {
        config.stats_pub_interval = read_time_with_mul(value);
    }
}

static inline void strip_spaces(char **str) {
    if (!*str) return;
    while (isspace(**str) && **str) ++(*str);
}

static inline void unpack_bytes(char **str, char *dest) {
    if (!str || !dest) return;
    while (!isspace(**str) && **str) *dest++ = *(*str)++;
}

int config_load(const char *configpath) {
    assert(configpath);
    FILE *fh = fopen(configpath, "r");
    if (!fh) {
        sol_warning("WARNING: Unable to open conf file %s", configpath);
        sol_warning("To specify a config file run sol -c /path/to/conf");
        return false;
    }
    char line[0xff], key[0xff], value[0xff];
    int linenr = 0;
    char *pline, *pkey, *pval;
    while (fgets(line, 0xff, fh) != NULL) {
        memset(key, 0x00, 0xff);
        memset(value, 0x00, 0xff);
        linenr++;
        // Skip comments or empty lines
        if (line[0] == '#') continue;
        // Remove whitespaces if any before the key
        pline = line;
        strip_spaces(&pline);
        if (*pline == '\0') continue;
        // Read key
        pkey = key;
        unpack_bytes(&pline, pkey);
        // Remove whitespaces if any after the key and before the value
        strip_spaces(&pline);
        // Ignore eventually incomplete configuration, but notify it
        if (line[0] == '\0') {
            sol_warning("WARNING: Incomplete configuration '%s' at line %d. "
                        "Fallback to default.", key, linenr);
            continue;
        }
        // Read value
        pval = value;
        unpack_bytes(&pline, pval);
        // At this point we have key -> value ready to be ingested on the
        // global configuration object
        add_config_value(key, value);
    }
    return true;
}

void config_set_default(void) {
    // Set the global pointer
    conf = &config;
    // Set default values
    config.version = VERSION;
    config.socket_family = DEFAULT_SOCKET_FAMILY;
    config.loglevel = DEFAULT_LOG_LEVEL;
    strcpy(config.logpath, DEFAULT_LOG_PATH);
    strcpy(config.hostname, DEFAULT_HOSTNAME);
    strcpy(config.port, DEFAULT_PORT);
    config.epoll_timeout = -1;
    config.run = eventfd(0, EFD_NONBLOCK);
    config.max_memory = read_memory_with_mul(DEFAULT_MAX_MEMORY);
    config.max_request_size = read_memory_with_mul(DEFAULT_MAX_REQUEST_SIZE);
    config.tcp_backlog = SOMAXCONN;
    config.stats_pub_interval = read_time_with_mul(DEFAULT_STATS_INTERVAL);
}

void config_print(void) {
    if (config.loglevel < WARNING) {
        const char *sfamily = config.socket_family == UNIX ? "Unix" : "Tcp";
        const char *llevel = NULL;
        for (int i = 0; i < 4; i++) {
            if (lmap[i].loglevel == config.loglevel)
                llevel = lmap[i].lname;
        }
        sol_info("Sol v%s is starting", VERSION);
        sol_info("Network settings:");
        sol_info("\tSocket family: %s", sfamily);
        if (config.socket_family == UNIX) {
            sol_info("\tUnix socket: %s", config.hostname);
        } else {
            sol_info("\tAddress: %s", config.hostname);
            sol_info("\tPort: %s", config.port);
            sol_info("\tTcp backlog: %d", config.tcp_backlog);
        }
        const char *human_rsize = memory_to_string(config.max_request_size);
        sol_info("\tMax request size: %s", human_rsize);
        sol_info("Logging:");
        sol_info("\tlevel: %s", llevel);
        sol_info("\tlogpath: %s", config.logpath);
        const char *human_memory = memory_to_string(config.max_memory);
        sol_info("Max memory: %s", human_memory);
        free((char *) human_memory);
        free((char *) human_rsize);
    }
}

```
<hr>

At this point we have all the pieces needed to have something running.
Let's add the last brick, a main:

<hr>
**src/sol.c**
<hr>

```c

#define _POSIX_C_SOURCE 2
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include "util.h"
#include "config.h"
#include "server.h"

int main (int argc, char **argv) {
    char *addr = DEFAULT_HOSTNAME;
    char *port = DEFAULT_PORT;
    char *confpath = DEFAULT_CONF_PATH;
    int debug = 0;
    int opt;
    // Set default configuration
    config_set_default();
    while ((opt = getopt(argc, argv, "a:c:p:m:vn:")) != -1) {
        switch (opt) {
            case 'a':
                addr = optarg;
                strcpy(conf->hostname, addr);
                break;
            case 'c':
                confpath = optarg;
                break;
            case 'p':
                port = optarg;
                strcpy(conf->port, port);
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
    // Override default DEBUG mode
    conf->loglevel = debug == 1 ? DEBUG : WARNING;
    // Try to load a configuration, if found
    config_load(confpath);
    sol_log_init(conf->logpath);
    // Print configuration
    config_print();
    start_server(conf->hostname, conf->port);
    sol_log_close();
    return 0;
}

```
<hr>

Short and clean, the project should now have all that is needed to work:

```bash
sol/
 ├── src/
 │    ├── mqtt.h
 │    ├── mqtt.c
 │    ├── network.h
 │    ├── network.c
 │    ├── list.h
 │    ├── list.c
 │    ├── hashtable.h
 │    ├── hashtable.c
 │    ├── server.h
 │    ├── server.c
 │    ├── trie.h
 │    ├── trie.c
 │    ├── util.h
 │    ├── util.c
 │    ├── core.h
 │    ├── core.c
 │    ├── config.h
 │    ├── config.c
 │    ├── pack.h
 │    ├── pack.c
 │    └── sol.c
 ├── conf
 │    └── sol.conf
 ├── CHANGELOG
 ├── CMakeLists.txt
 ├── COPYING
 └── README.md
```

That's a moderate amount of code to manage, for compilation I'd like to write
custom `Makefile` usually, but this time, as shown on the folder tree, I'll
consider using a `CMakeLists.txt` defined template and `cmake` to generate it;
probably not the advisable as `cmake` is such a huge piece of software to generate
artifacts for such a small and dependency-free software, however I was curious to
see how it worked and took the moment:

```bash

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

```

The only part worth a note is the DEBUG flag that I added, it makes cmake
generate a different `Makefile` that compile the sources with some additional
flags to catch and signal memory leaks and undefined behaviours.

So the next move is to generate the `Makefile`

```bash

$ cmake -DDEBUG=1 .

```

and compile the sources:

```bash

$ make

```

This will produce a *sol* executable that can be used to start the broker,
supporting a bunch of parameters and flags we handled in main function.<br>
We run the system with the **-v** (verbose) flag, to have all debug logging
printed out on screen to better follow the execution.

```bash

$ sol -v

```

And that's it for now, there are probably a lot of bugs, memory leaks that must
be fixed and redundant code, as well as messy includes, but the skeleton is all
there and have to be considered an MVP. Part-7 will come soon, with testing and
some snippets using `paho-mqtt` to play with the newborn. Of course some tests
would be added, at least for the main features. For now you can enjoy the
[bonus part](../sol-mqtt-broker-bonus) adding concurrent capabilities to the
system.
