---
layout: post
title: "Sol - An MQTT broker from scratch. Part 5 - Handlers"
description: "Writing an MQTT broker from scratch, to really understand something you have to build it."
tags: [c, unix, tutorial]
---

This part will focus on the implementation of the handlers, they will be mapped
one-on-one with MQTT commands in an array, indexed by command type, making it
trivial to call the correct function depending on the packet type.

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
