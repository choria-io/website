+++
title = "Federations of Collectives"
icon = "<b>5. </b>"
+++

Deploying large distributed Collectives can be tricky mainly due to the operation concerns of running a single huge Middleware Deployment spanning multiple Data Centres or even Countries.

In the past there was no choice but to run such a single distributed Middleware deployment. With Choria you can divide your network into smaller, managable, parts and use a component called a *Federation Broker* to combine them into a single entity.  With the NATS middleware Federation is recommended for setups spanning multiple Data Centres as it only supports Full Mesh networks.

## Topology

Below you can see an overview of a Federation of Collectives.  The locations *london*, *tokyo* and *new york* are completely standalone and isolated from a Middleware Perspective.

The central Middleware act as a management location where all the Collectives are Federated into one.  The Collectives can not communicate with each other but the Federation can communicate with them all thus creating a safe way to manage Development and Production without exposing Production to Development.

![Federation of Collectives](../choria_federation.png)

## Benefits

 * Easier to manage at a large scale
 * Member Collectives remain functional as standalone entities
 * The scale and throughput is increased thanks to offloading some functionality from client to the Federation Brokers
 * All MCollective features remain functional
 * A Collective can belong to several Federations
 * Federation Brokers need no shared infrastructure like Consul or databases

## Terminology

### Member Collective

A traditional standalone MCollective install with a single cluster of Middleware, typically regional or per DC

### Federation of Collectives

A network created by combining many standalone Collectives into one unit.  The Federation can communicate with all member Collectives, Collectives cannot communicate with each other.

### Federation Broker

A cluster of software components load sharing the work of brokering messages between a Collective and a Federation of Collectives.  Individually the cluster daemons are called Federation Broker Instances.
