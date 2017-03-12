+++
title = "Federations of Collectives"
toc = true
weight = 225
+++

Choria allows you to configure *Federations of Collectives* which allows a client to
communicate with many independant collectives.

{{% notice warning %}}
This is a new feature that has not yet shipped, the documentation serves as a preview for discussion.
{{% /notice %}}

![Federation of Collectives](../../choria_federation.png)

## Motivation

Ideally you have a cluster in every region where the regional nodes connect.  The regional
mesh connect to a central meshed cluster, this was possible with ActiveMQ and RabbitMQ but
not with NATS.

Even with ActiveMQ and RabbitMQ though such clusters are problematic as the subscriptions
and dupe tracking quickly add a lot of requirements on your network.

Configuring huge multi region meshes with those brokers are very complex and not well
documented.

NATS only support full mesh networks which is not suitable for large multi region setups.

Choria Federations utlize a MCollective protocol aware cluster of bridges that connects
many collectives to a *Federation of Collectives*.

## Features

  * Easy to configure with only 1 setting in MCollective
  * Greatly simplify the management of multi region collectives
  * Member Collectives remain functional as standalone entities
  * Member Collectives can be members of several Federations
  * Federation Brokers are clusterable as they are completely stateless
  * Increases the scale and throughput of direct addressing by off loading the
    creation of per target message to the Federation Brokers - load shared across
    the Federation Broker Cluster.
  * All features of MCollective remain functional including sub collectives,
    which then can span the entire federation.

## Limitations

  * You need to use the same CA everywhere.  This is planned to be changed in future.
  * You cannot connect a client to a Federation Broker that in turn connect to a
    Federation Broker.  Federation Brokers connects Federations to Collectives only.
  * All messages go to all Collectives, this is as usual for Broadcasts. Direct Addressed
    messages though will be duplicated between collectives.  NATS though is very efficient
    at ignoring messages with no subscriptions and drops them at a rate of 10s of millions
    a second so I do not anticipate this to be a problem.

## Terminology

### Collective

A standalone MCollective network with it's own isolated NATS cluster.

### Federation

A collection of Collectives that a correctly configured client can access
as one.  A federation has it's own isolated NATS cluster.

### Federation Broker

A daemon that takes care of connecting a Collective to a Federation.
The daemon is stateless and supports being clustered by just running
more with the correct configuration.

## Configuring

Configuring the individual Collectives is exactly as normal there is nothing special
to do in any individual Collective.

To configure your client to belong to a Federation create the following hiera data:

```yaml
mcollective::client_config:
  "choria.federation.collectives": "tokyo, london, new_york"
```

A client that is configured as above will communicate with all those Collectives, you
can limit it to a specific Collective using something like:

```bash
$ CHORIA_FED_COLLECTIVE=china mco rpc rpcutil ping
```

This will override the *choria.federation.collectives* setting but will also instruct
a client without this setting to become a federated client.

## Federation Broker

To be completed
