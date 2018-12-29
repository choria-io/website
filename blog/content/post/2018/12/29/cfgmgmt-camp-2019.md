---
title: "CfgMgmtCamp 2019"
date: 2018-12-29T13:28:00+01:00
tags: ["talks"]
---

I will be giving a talk at the 2019 installment of [CfgMgmtCamp](https://cfgmgmtcamp.eu/) in Ghent on 4 to 6th February 2019

The talk will be focussed on [Choria Data Adpaters](https://choria.io/docs/adapters/), [NATS Streaming](https://github.com/nats-io/nats-streaming-server), metadata and will discuss the design of the [Choria Stream Replicator](http://github.com/choria-io/stream-replicator).

I'll hopefully also show off something new I've been hacking on on and off!

The CFP submission can be seen below the fold, I hope to see many Choria users there!

<!--more-->

## Solving large scale operational metadata problems using stream processing

Stream processing technologies are usually associated with social media, click streams, machine learning and big data however this talk will show it can be useful in solving difficult problems operations face when dealing with large server estates.

A very large scale stream processing pipeline have been built managing metadata for 100s of thousands of nodes using technologies from Choria.io and NATS.io, this talk will explore the design, performance and potential uses of such a platform.
Areas covered:

 * basic streaming overview
 * design of the global network
 * design of a replication tool particular to the needs
 * building hierarchical cached views of the estate
 * exposing data to other systems with examples
 * using stream processing methodologies to perform real time analysis and remediation of the fleet

We’ll also explore how this is a good parallel for IoT where the mentioned node scale is considered small.

While this talk demonstrates a platform built using specific technologies and the design of those will influence the talk, it’s not a vendor talk and we’ll make efforts so the ideas are transferred and not the glossies from any one tool.