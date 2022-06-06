+++
title = "Related Projects"
weight = 207
toc = true
+++

There are a number of related projects in the project GitHub repositories, I'll call out a few key ones here:

## Choria Plugins

A number of Ruby based plugins are maintained in the [Choria Plugins](https://github.com/choria-plugins) organisation, these provide features such as Package, Service and Puppet management.  Many of them are installed by default when you install Choria but it's worth a check what else is there.  They are all distributed as [Forge Modules](https://forge.puppet.com/choria).

## Stream Replicator

Choria uses technology from [NATS.io](https://nats.io) in its Network Broker, [Choria Streams](/streams/) is a managed instance of [NATS JetStream](https://docs.nats.io/jetstream/).  The [Data Adapters](../../adapters/) supports bridging Registration and Choria Replies onto a Choria Streams service.

You might have multiple data centers and want to transport these registration data to a central location in a way that is resilient to network outage and would support building multiple caches of data in multiple locations with different freshness cadences. You can also subscribe entirely different kinds of processor for the data and each processor can consume the data at its own pace.

This is a key component to be able to build scalable asynchronous REST systems, schedulers and more.

The [Choria Stream Replicator](https://github.com/choria-io/stream-replicator) is the tool to achieve this and other tools mentioned below are developed to be compatible with data flows managed using it.

## Choria Asyncjobs

Think of it as a distributed share-nothing Cron system.  You can execute jobs with workers written in any language and hosted in Kubernetes, Docker or elsewhere.

It is an Asynchronous Job Queue system that relies on NATS JetStream for storage and general job life cycle management. It is compatible with any NATS JetStream based system like a private hosted JetStream, Choria Streams or a commercial SaaS.

Each Task is stored in JetStream by a unique ID and Work Queue item is made referencing that Task. JetStream will handle dealing with scheduling, retries, acknowledgements and more of the Work Queue item. The stored Task will be updated during the lifecycle.

Multiple processes can process jobs concurrently, thus job processing is both horizontally and vertically scalable. Job handlers are implemented in Go with one process hosting one or many handlers. Other languages can implement Job Handlers using NATS Request-Reply services. Per process concurrency and overall per-queue concurrency controls exist.

See its [documentation](https://choria-io.github.io/asyncjobs/) for more details.

## Choria Appbuilder
To a large extent these are tribal knowledge and something that is a big hurdle for new members of the team. The answer is often to write wiki pages capturing run books that has these commands documented.

This does not scale well and does not stay up to date.

What if there was a CLI tool that encapsulated all of these commands in a single, easy to use and easy to discover command.

The appbuilder project lets you build exactly that by specifying a model for your CLI application in a YAML file and then building custom interfaces on the fly.

See its [documentation](https://choria-io.github.io/appbuilder/) for more details, we also have a [video introduction](https://www.youtube.com/watch?v=wbu3N63WY7Y).

## Choria Server Provisioner

In large dynamic environments where you do not have configuration management systems like Puppet or wish to do custom CA integration we provide a provisioning system that gives you full control over the life cycle of the Choria Daemon, unconfigured daemons enter a provisioning state where a software component can reach out to them and configure them.

This is an advanced feature and requires custom builds to be made (but we provide the tools for this).

See the documentation in [provisioner](https://github.com/choria-io/provisioner).

A video demonstrating this capability can be seen below:

<iframe width="840" height="473" src="https://www.youtube.com/embed/7sGHf55_OQM" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

## Helm Charts

As part of the Scout project we are working on a fully [Kubernetes hosted solution](https://github.com/choria-io/helm) that's independent of Puppet. It auto enrolls nodes, auto configures them and manage their life cycle completely without needing any external components. 
