+++
title = "Middleware"
weight = 30
+++

Choria communicates almost exclusively over the middleware, this document describes the target names and messages you might find on the middleware. You might use this for debuging or learning purposes using the `choria tool pub` and `choria tool sub` commands.

The Choria broker is a managed instance of the [NATS Server](https://github.com/nats-io/gnatsd), target syntax shown here matches the standards it support.

## Core RPC Messages

### Unfederated

This is the core RPC related message targets, this is how machines discover nodes and communicate with them either 1:1 or 1:n.

These are namespaced on `collective` allowing one to make something like a VLAN - where servers and clients in a particular `sub collective` can only find others configured with the same.

|Target|Description|Schema|
|------|-----------|------|
|`collective.reply.>`|Replies to requests made by clients, example `mcollective.reply.dev1.example.net.c2a764e6013a44adb848904ff7d74ff4`|[choria:transport:1](https://choria.io/schemas/choria/protocol/v1/transport.json)|
|`collective.broadcast.agent.>`|Requests from clients to specific agents broadcasted to all servers interested|[choria:transport:1](https://choria.io/schemas/choria/protocol/v1/transport.json)|
|`collective.node.>`|Requests from clients to specific nodes regardless of the agent aka `directed`|[choria:transport:1](https://choria.io/schemas/choria/protocol/v1/transport.json)|

### Federated

Federation is a router between different networks, when a client makes a request it publishes to a specific address instead of the usual ones, the Federation Broker will then publish to the above targets on the users behalf.

Likewise replies are received on a different target and the Federation Broker will send them to the usual client locations on the servers behalf.

These are namespaced on `collective` here, each Federation `Member Collective` has a unique name.

|Target|Description|Schema|
|------|-----------|------|
|`choria.federation.collective.collective`|When federated, replies to clients from server that would typically go to `collective.reply.>` are directed here instead|[choria:transport:1](https://choria.io/schemas/choria/protocol/v1/transport.json)|
|`choria.federation.collective.federation`|When federated, requests to servers that would typically go to `collective.broadcast.>` or `collective.node>` goes here instead|[choria:transport:1](https://choria.io/schemas/choria/protocol/v1/transport.json)|

