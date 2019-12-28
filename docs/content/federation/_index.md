+++
title = "Federations of Collectives"
pre = "<b>6. </b>"
weight = 60
+++

Federations of Collectives combines multiple standalone Collectives into one.  This improves the operability of large distributed networks and greatly increase the scale achievable.

Deploying large distributed Collectives can be tricky mainly due to the operation concerns of running a single huge Middleware deployment spanning multiple Data Centers or even Countries.

In the past there was no choice but to run such a single distributed Middleware deployment. With Choria you can divide your network into smaller, manageable, parts and use a component called a *Federation Broker* to combine them into a single entity.  If your network span multiple data centers Federating your networks using the Federation Broker is recommended.

## Topology

Below you can see an overview of a Federation of Collectives.  The locations *london*, *tokyo* and *new york* are completely standalone and isolated from a Middleware Perspective.

The central Middleware acts as a management location where all the Collectives are Federated into one.  The Collectives can not communicate with each other but the Federation can communicate with them all thus creating a safe way to manage Development and Production without exposing Production to Development.

![Federation of Collectives](../choria_federation.png)

The Federation Brokers are a share-nothing architecture. They are completely stateless and scale horizontally and vertically.

## Benefits

 * Easier to manage at a large scale
 * Member Collectives remain functional as standalone entities
 * The scale and throughput is increased thanks to offloading some functionality from the Client to the Federation Brokers
 * All Choria features remain functional
 * A Collective can belong to several Federations
 * Federation Brokers need no shared infrastructure like Consul or databases
 * Extensive Prometheus metrics about the internals of the brokers

## Terminology

### Member Collective

A standalone Choria install with a single cluster of Middleware, typically regional or per DC

### Federation of Collectives

A network created by combining many standalone Collectives into one unit.  The Federation can communicate with all member Collectives, Collectives cannot communicate with each other.

### Federation Broker

A cluster of software components load sharing the work of brokering messages between a Collective and a Federation of Collectives.  Individually the cluster daemons are called Federation Broker Instances.
