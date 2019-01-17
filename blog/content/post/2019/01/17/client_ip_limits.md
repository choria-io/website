---
title: "Limiting Clients to IP Ranges"
date: 2019-01-17T13:28:00+01:00
tags: ["security"]
draft: false
---

The upcoming set of releases have a strong focus on security.  We will introduce a whole new way to build centralized AAA if a site desires and a few smaller enhancements.  One of these enhancements is the ability to limit where clients can be used on your network.

Today the security model allow anyone with the correctly issued and signed certificates to make client requests from anywhere on your network. This is generally fine as the certificates are not to be shared, however there are concerns that there might be rogue clients on your network perhaps outside of your update strategy or just as a form of shadow orchestration system. You could also have concerns about the fact that using just a server certificate one can read all replies the entire network sends that might contain sensitive information.

If you have this concern the upcoming version `0.10.0` of the Choria Broker will include the ability to limit what networks clients can come from.

<!--more-->
## How we use the middleware

Choria has a number of network targets that it uses during its general life, we want to prevent everyone from having to know how this work so the managed allow/deny lists takes care of restricting just how Choria actually uses the network.

We documented all the targets that Choria uses on the network in a new [documentation page](https://choria.io/docs/development/middleware/).

## Configuration

```ini
plugin.choria.network.client_hosts = 192.168.0.1, 192.168.20.0/24, 2a01:7e00::/64
```

With this single line you're allowing clients - and by extension also Federation Brokers - only on those networks.

This is achieved by allowing clients to do anything on the network, while servers are restricted.

One might convincingly argue that we should do this using a default deny list rather than a default allow list but I think there are legit cases where other software will live on the Choria Broker network - like NATS Streaming Servers.  Additionally custom targets like where registration is published and messages directed at the Adapters are basically free form.

To keep the configuration simple we went the default allow route. The alternative would require users to write a full set of allow/deny rules which might be beyond the scope of the managed NATS service in the Choria Broker, you could always run a full NATS Server if you are so inclined.

### Subscribe limits

We limit what servers can subscribe to as follows, here mainly we want to limit what sensitive information can be learned by using a server certificate and a NATS client:

|Target|Action|Description|
|------|------|-----------|
|`*.reply.>`|Deny|Cannot subscribe to replies, cannot read replies from other nodes|
|`choria.federation.>`|Deny|Cannot read messages on any federation targets, so cannot intercept replies or requests|
|`choria.lifecycle.>`|Deny|Cannot read any lifecycle events from anyone else, so cannot learn what other nodes are out there|
|`>`|Allow|Allow all other subscription requests|

### Publish Limits

We limit where servers can publish to prevent them from making requests to other nodes - though their certificates are not capable of making requests anyway - but they might be able to effect a DOS or you might just have rogue clients where you did not expect to have them.

|Target|Action|Description|
|------|------|-----------|
|`*.broadcast.agent.registration`|Allow|Allows the default registration target setup|
|`*.broadcast.agent.>`|Deny|Cannot make any broadcast requests|
|`*.node.>`|Deny|Cannot make any directed requests|
|`choria.federation.*.federation`|Deny|Cannot initiate any requests to the federation brokers|
|`>`|Allow|Allow all other publish requests|

Note publishing to `choria.federation.*.collective` is allowed as that's where federated replies will go.

## Conclusion

This is probably an advanced feature, we think the certificate based model is suitable to the usage model but in some strict environments this might not be enough.  We tried to keep the setup as simple as possible leaving full manual setup to users who can run their own full blown NATS Servers.
