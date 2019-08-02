---
title: "NATS 2.0 Based Broker"
date: 2019-08-01T9:00:00+01:00
tags: ["nats", "network"]
draft: false
---

It's been a few months since our last releases - usually we release monthly but there's been a delay, today we'll give you some background on why.

We've been very hard at work on adopting [NATS 2.0](https://github.com/nats-io/nats-server) for the Choria Network Broker. NATS 2.0 was released 5th of June 2019 and represents a huge improvement in capabilities.

The major feature is around security, NATS is now multi tenant capable which means you can create secure isolated collectives in a Choria Network Broker.  This is very exciting since our *subcollective*, while still relevant, was always quite hard to secure.

Additionally there have been a huge shift in networking capabilities allowing new ways to form super clusters and to extend your broker foot print globally. This will allow us to look toward other models of federation rather than our own Federation Brokers.

If you read further you will find much have changed and many new features are available, however for the typical use case nothing has to change.  **Everything keeps working and even old clients and agents will continue to function with no configuration of behaviour change**.  If you do not use these features or do not need them, there is nothing new.

The challenge for us in integrating NATS 2.0 into the Choria Network Broker is to map these new capabilities onto Choria use cases and make them configurable in a way that make it comfortable within Choria. 

Completing this and making everything aware of these new features is a big undertaking, we've been focussing on the network broker in the present push. The code is successfully running on some of my smallest clusters - around 10k nodes - and where we identified problems the NATS team have been quick to help stabilize.

As always you can expect when we release this we'll certify it to work out of the box on 50 000 nodes. As a bonus we have found that the new Network Broker is quite a bit faster than the existing one.

Read on for the gory details!
<!--more-->
## Security Improvements

## Accounts

NATS 2.0 allow you to create something akin to a chain or trust in x509.

Once this is enabled all clients connecting to the network needs credentials and all clients belong to an account. You can have just one account that's fine, but if you have many they are securely isolated.

There are a number of entities to think about:

 1. The *Operator* runs the network, this is kind of like a Root CA, who controls the entire network. An operator can create new networks or allow new entities into the multi tenant network.
 1. An *Account* in Choria would be something like a business unit, or a data center or a business group.
 1. A *Client* belongs to an *Account* and represents each managed machine or Choria client.

Connections made by a *client* are authenticated using a credential, the credential belongs to an account and any client within an account can communicate with each other.

This is the underlying feature that all the rest is built on. It allows you to offer Choria as a service with many tenants cohabiting, it allows data isolation for stronger auditing features and better global networks. By connecting our CA, our Provisioning, our over the air updates and automatic user enrollment to this we can build a next generation secure orchestration platform suitable for building cloud backplanes.

Configuring accounts in Choria is quite easy:

```nohighlight
plugin.choria.network.operator_account = OPS
plugin.choria.network.system_account = ADWJVSUSEVC2GHL5GRATN2LOEOQOY2E6Z2VXNU3JEIK6BDGPWNIW3AXF
```

This tells the broker that it is operated by the entity *OPS* - which would define accounts etc. And that within that entity is a System Account identified by the public key mentioned (below more about those).

There's a utility to create the keys to place in a well known directory, from there the whole feature is enabled.  Nearer to release we'll document this all. You can read more about accounts in the [NATS Documentation](https://nats-io.github.io/docs/nats_server/jwt_auth.html).

## Per client security rules

When issuing the client credential you can embed in them access lists, in Choria that means we can quite strictly configure a credential for a user - machines would have tight credentials preventing them from being clients and visa versa. This will be much more specific and secure than the current feature in Choria Broker that attempts to create one size fits all ACL rules.

The Choria Provisioner will support issuing credentials for Clients with appropriate access lists embedded. It will support joining unprovisioned machines into a specific account and so forth.

There are some additional things we can support if we find a good use case like expiry times, payload sizes and message counts.

## Cross account communication

You can enable cross account communication, this will be used in Choria for node registration data and our current Data Adapters will support these.  You could have 10 accounts (business units, tenants etc) but you might want to receive centrally all their metadata to store etc.

One can embed in the accounts rules allowing the registration data published by *node1* in account *Acme* to be received by user *cmdb* in account *OPS*.  Thus a single pool of data processors can serve any number of tenant accounts without having to provision per account data processors.

## Networking Improvements

The networking improvements below are all compatible with Accounts/Multi Tenancy and our adoption is to support a massive global scale network like you would need for a managed IoT network consisting of millions or tens of millions of devices.

I hope to publish a few more posts soon showing scenarios where you would use these features. These were not built by Choria, they are core NATS features, you can use them via NATS 2.0 standalone to build your own tools as well.

### Gateways

Choria Broker supports clustering really easily with automatic support for TLS and everything.  The clusters being NATS based forms a full mesh networking and so we think a sweet spot for these size wise is below 5 nodes. They are optimized for a LAN and should not span WAN links.

Gateways allow you to create a connection between 2 Clusters and traffic will leave the local cluster only when necessary based on interest.

Long time users would remember that we could support similar architecture with ActiveMQ where one could create a *subcollective* for, lets say, *NewYork* and *Tokyo* while there could be a over all *subcollective* joining both together. Network traffic would only leave the local network if nodes in other DCs are directed at.

Gateways let us do the same, are multi tenant aware and transparent to the application. 

You will soon be able to link 2 isolated networks *NewYork* and *Tokyo* together as simple as this:

```nohighlight
plugin.choria.network.gateway_port = 7001
plugin.choria.network.gateway_name = NewYork
plugin.choria.network.gateway_remotes = Tokyo,NewYork
plugin.choria.network.gateway_remote.Tokyo.urls = c1.tokyo.example.net:7001,c2.tokyo.example.net:7001
plugin.choria.network.gateway_remote.NewYork.urls = c1.nyc.example.net:7001,c2.nyc.example.net:7001
```

We'll now have the 2 data centers connected and only traffic that needs to traverse the network boundaries will do so.  As with clusters this will all automatically be secured with TLS based on the same rules as always, it also support custom TLS certificates for your own CA infrastructure.

Further information about Gateways are in the [NATS documentation](https://nats-io.github.io/docs/gateways/).

### Leaf Nodes

Imagine you have different networks for 2 business units and they are hosted on the same Choria Broker in different accounts. You would want to enroll new nodes into the correct network but prior to enrollment you need to isolate these unenrolled machines.

When you provision a new node without yet any certificates etc you will need to connect to the provisioning network in the clear since you do not yet have TLS certificates. The provisioning process is designed to be secure even in the clear. But the problem is how to join these unknown machines to a specific account that lives in a TLS secured Broker cluster?

Leaf nodes allow you to set up a Choria Broker that could have a different authentication method - no TLS for example - and would force all connections to it into an isolated network like a VLAN would. The Leaf node would connect to your cluster and from there allow access to everything in the cluster that is in the account the leaf node belongs to.

This is ideal for the provisioning scenario above or for allowing remote RPC clients connect to your network without having to do too much provisioning of certificates and more.

You will soon be able to add a Leaf Node to a Choria Network quite easily, here we connect to the OPS remote using a file with credentials and the all connections to the leafnode will be forced into the account:

```nohighlight
plugin.choria.network.leafnode_port	= 7002
plugin.choria.network.leafnode_remotes = OPS
plugin.choria.network.leafnode_remote.OPS.url = c1.ops.example.net:7002
plugin.choria.network.leafnode_remote.OPS.credential = /etc/choria/leafnodes/ops.creds
plugin.choria.network.leafnode_remote.OPS.account = ADMB22B4NQU27GI3KP6XUEFM5RSMOJY4O75NCP2P5JPQC2NGQNG6NJX2
```

Further information about Leaf nodes are in the [NATS documentation](https://nats-io.github.io/docs/leafnodes/).

## Observability Improvements

If you enable a System Account in your Choria Network Broker as above you will be able to passively observe the network using a NATS client, for example:

```nohighlight
$ choria tool sub "$SYS.ACCOUNT.*.CONNECT"
```

Will show notifications of every connection received on every account.  There's a wealth of events published by the Broker, see more at the [NATS Documentation](https://nats-io.github.io/docs/sys_accounts/sys_accounts.html).

Using this we can create observability tools that create live graphs of connections on a per account basis or CLI tools to actively observe the state of the network.