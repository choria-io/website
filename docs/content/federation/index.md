+++
title = "Federations of Collectives"
icon = "<b>5. </b>"
+++

Deploying large distributed Collectives can be tricky mainly due to the operation concerns of running a single huge Middleware Deployment spanning multiple Data Centres or even Countries.

In the past there was no choice but to run such a single distributed Middleware deployment. With Choria you can divide your network into smaller, managable, parts and use a component called a *Federation Broker* to combine them into a single entity.  With the NATS middleware Federation is recommended for setups spanning multiple Data Centres as it only supports Full Mesh networks.

{{% notice tip %}}
This feature is availble since 0.0.25
{{% /notice %}}

## Topology

Below you can see an overview of a Federation of Collectives.  The locations *london*, *tokyo* and *new york* are completely standalone and isolated from a Middleware Perspective.

The central Middleware act as a management location where all the Collectives are Federated into one.  The Collectives can not communicate with each other but the Federation can communicate with them all.

![Federation of Collectives](../choria_federation.png)

## Benefits

 * Easier to manage at a large scale
 * Member Collectives remain functional as standalone entities
 * The scale and throughput is increased thanks to off loading some functionality from client to the Federation Brokers
 * All MCollective features remain functional
 * A Collective can belong to several Federations
 * Federation Brokers need no shared infrastructure like Consul or databases
