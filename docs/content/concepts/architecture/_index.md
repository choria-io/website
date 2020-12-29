+++
title = "Architecture"
weight = 202
toc = true
icon = "<b>1. </b>"
+++

## The Choria Components

Choria is a client-server model system. A **Server** exposes a set of APIs that allow clients to manage the server - think install a package, get the status of a service or trigger a Puppet run. A **Client** instructs one or 10s of thousands of servers to perform one of the actions that they expose via their API.

The architecture of Choria is based around three main components: servers, clients, and the middleware broker.

![Client Server Overview](../../basic_client_server_overview.png)

## Servers

A Choria Server is a node that is controllable by a Choria client.  Each server runs an instance of *choria server*.

These servers primarily host **Agents** that make up the APIs that the server expose. You might have a *package*, *service* and *puppet* agent but also your own *deploy* agent or any other thing you want.

Each agent has a number of **Actions**, imagine an agent dedicated to package management might have *install*, *update*, *remove* and *status*.  These are all **Actions**.

Each Choria Server has a unique SSL certificate, typically - and by default - it uses the ones Puppet creates and manages as part of its life cycle.

Nodes perform Authentication Authorization and Auditing on every request.

Servers do not need additional open ports, nothing will listen for connections, they make a connection out to the middleware and can discover where the right middleware is using SRV records.

Servers can host **Autonomous Agents** that continuously manage nearly anything you wish using a Control Loop (like a Kubernetes Operator). This is foundational technology of our **Scout** monitoring system.

## Clients

A Choria Client is any piece of software that can request servers perform some action. Typically you will interact with Choria using the *mco* or *choria* CLI tool but there is also a rich API allowing you to write automated systems in Ruby or Golang.

Each client has its own unique SSL certificate, typically signed by your Puppet CA.

We have a generic client called *choria rpc* that constructs a user interface dynamically using the introspection abilities exposed from Choria, but one might also write custom clients - often called *applications* - to create custom user interfaces when the auto generated one is not suited.

Clients do not need additional open ports, nothing will listen for connections, they make a connection out to the middleware and can discover where the right middleware is using SRV records.

## Middleware Brokers

Choria provides a system called Choria Broker.  This Broker is to Choria what Ethernet is to nodes. All communications with the outside world will traverse this broker.

Servers and Clients will open connections to the Broker, all communication with nodes will happen via this.

The Choria Broker is very light weight and performant.  On a 4GB memory compute node you will be able to handle communication needs for 50 0000 Servers.

The Choria Broker can be clustered on a LAN level and we support Federating many geographically distributed networks into one giant network using the Choria Federation Broker.

A Streaming Server is currently in Technology Preview and will form a basis for persisting node data such as registration, monitoring and life cycle events.

![Federation Overview](../../choria_federation.png)

## Data Adapters

The Choria Broker provides a system that can receive regular data from your network and adapt it to other systems.

Imagine your nodes publish their metadata such as Puppet Facts every 10 minutes (something the Choria Server can do out of the box), the Data Adapter will live in all your Choria Broker and translate that data onto other systems like [NATS Streaming](https://github.com/nats-io/nats-streaming-server) where your team can write software to process this data in a streaming fashion.

![Adapters Overview](../../adapters-overview.png)

Support for Kafka, Kinesis and other similar systems is planned.
