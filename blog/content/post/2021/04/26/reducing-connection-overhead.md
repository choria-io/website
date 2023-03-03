---
title: "Reducing connection overhead for branch office scenarios"
date: 2021-04-26T00:00:00+01:00
tags: ["leafnodes"]
draft: false
author: Romain Tartière
---

Because Choria allows you to manage nodes spread all around the world, and because you might be working from your laptop, far away from the (bad) Wi-Fi access points that connects you through (bad) PLC to the (bad) internet connection from the (not bad) island you are on, you may experience inconvenient latency and unreliabilities.

The reason is quite simple: while the Choria servers maintain a permanent connection with the message broker, the Choria client has to establish a new connection with the middleware for each request.  Latency and packet loss do not help with establishing TLS encrypted connections in a timely fashion.

But _good news everyone_!  NATS — the messaging system Choria is built on — has built-in support for so-called _[leaf nodes](https://docs.nats.io/nats-server/configuration/leafnodes)_ which offer a solution to this problem.

<!--more-->

A _leaf node_ acts as a broker node.  It accepts client connections on port 4222 and allow messages to be distributed to the server nodes.  The key point is that it maintains a permanent connection with the Choria broker, so when a client connect to it, it is immediately able to send messages through the middleware.

![](/blog/mom/leafnode.png)

Installing a leaf node nearby a Choria client (on the LAN in the office if multiple employees need to access the message broker, or maybe directly on your laptop if you happen to move around) allows the Choria clients to establish the expensive TLS handshake with a node next to them rather than one far far away, making the whole process faster and more reliable.

Let's see how to proceed to deploy a leaf node.

## Deploying a leaf node

The choria-choria Puppet module recently gained the capacity to configure leaf nodes ([#239](https://github.com/choria-io/puppet-choria/pull/239)).  It is part of the [April 2021 Releases](/blog/post/2021/04/23/april_releases/) of Choria and require at least Choria Server version 0.22.0.  This section shows how to proceed.

Before starting, you might want to measure the latency when you start to compare with what you have at the end.  Let's use `choria ping` for this purpose:

```
romain@zappy % choria ping
[...]
---- ping statistics ----
47 replies max: 378ms min: 172ms avg: 304ms overhead: 1.940s
```

The _max_ / _min_ / _avg_ figures are not the one we are interested in here because they measure the latency between the choria client and the choria servers.  The _overhead_ is what we want to improve: the time needed by the client to establish a connexion to the broker node, nearly 2 seconds in my case.

### On Choria broker nodes

Let's start with the broker nodes configuration.  It must be adjusted in order to accept connections from leaf nodes on a dedicated port:

```
choria::broker::leafnode_port: 7422
```

### On Choria leaf nodes

A _leaf node_ is really just a broker.  We have to provide it the list the broker nodes it can use to reach the actual middleware (on the ports we have configured in the previous step) and it will listen on port 4222 to accept client connections:

```
choria::broker::leafnode_upstreams:
  choria:
    url: "nats://choria1.example.net:7422,nats://choria2.example.net:7422,nats://choria3.example.net:7422"
```

### On Choria client nodes

Choria clients can now connect to the leaf node instead of the choria broker to join the middleware.  Adjust the `plugin.choria.middleware_hosts` setting accordingly:

```
mcollective::common_config:
  plugin.choria.middleware_hosts: "leafnode.example.com:4222"
```

We are now done!

### Checking the result

Let's run `choria ping` one more time to compare the results.  The first thing you will notice is that you will get a flow of replies almost immediately.

```
romain@zappy % choria ping
[...]
---- ping statistics ----
47 replies max: 509ms min: 171ms avg: 318ms overhead: 47ms
```

The latency timings are now slightly higher (in my case from 304 to 318 ms, so a 4.6% increase of the average latency) because messages now have to go through one more layer between the client and servers.

But the key point: the _overhead_ was reduced from 1.940s to 47 ms, about 2.4% of the initial duration.
