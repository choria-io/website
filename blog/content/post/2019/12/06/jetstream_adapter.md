---
title: "New JetStream Adapter"
date: 2019-12-06T9:09:00+01:00
tags: ["adapter"]
draft: false
---

The upcoming Choria Server release will support publishing fleet data to JetStream a new Streaming Server from the folks who built NATS - the core technology of the Choria Broker.

This technology is currently in tech preview and so we've added early support for it so Choria users can test it for their needs.

Support for JetStream will be in the nightly build tonight and a release will be soon.

Read on for some background on the why and how.

<!--more-->
At present we support receiving node metadata from a fleet of nodes and placing it into a [NATS Streaming Server](https://docs.nats.io/nats-streaming-concepts/intro). I've used this to great effect to move vast amounts of metadata around the world with the help of the [Choria Stream Replicator](https://github.com/choria-io/stream-replicator).

Using these two tools I've built a system that will maintain a real time view of 100s of thousands of fleet nodes with 5 minute resolution including node down detection in a highly distributed and stable manner.

NATS Streaming though is a separate product from the core NATS server and has it's own client libraries and turns the model on the head a bit which have always made it a bit weird to explain to others. 

The NATS team have for a long time now been working on a next generation server called JetStream which integrates with core NATS - and core client libraries - and have many new features that would make it suitable for work queues in addition to the more traditional use while being secure, multi tenant and soon clustered.

Yesterday it was [announce that JetStream has entered Tech Preview Status](https://github.com/nats-io/nats-server/blob/jetstream/jetstream/README.md) so to assist with testing it I added an adapter that will let Choria place data in it using the same model as with NATS Streaming today.

Basic configuration and behavior is familiar to those using the NATS Streaming adapter today, see the [official documentation](https://choria.io/docs/adapters/jetstream/) for details.