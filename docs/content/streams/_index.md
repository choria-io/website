+++
title = "Streams"
pre = "<b>8. </b>"
weight = 80
+++

Choria Streams is a managed instance of [NATS JetStream](https://docs.nats.io/jetstream) embedded in Choria Broker.

Streams provide persistence for messages published on the NATS network, once messages are persisted to disk they can
later be consumed by one or multiple consumers. These consumers can process the data, do calculations, analyze, train
ML models or any similar actions in any of the 40+ language that support NATS.

Messages are stored in a WAL on disk and various message expiration models are supported.

We are in the process of developing our use of this technology, so some features are still embryonic and being developed.

Today we support:

 * Auto Configuring Streams for Events, Autonomous Agent Data and Scout Data 
 * Highly available, replicated share-nothing, storage
 * Publishing data from the Registration system to Choria Streams using [Adapters](../adapters/)
 * Constraining concurrency of some tasks in Autonomous Agents and cron using [Choria Governor](governor/)
 * Sending data from fleet nodes into a stream via the [Stream Submission](../submission) system
 * Scout check history viewer based on historical data for example *choria scout watch --history 5m*

Planned:

 * High performance distributed key-value store integrated into Autonomous Agents
 * Publishing Choria Audit logs into a central store
 * Leader elections to bring HA to Tally, Provisioner and Stream Replicator
 * Stream Replicator support for Choria Streams

The data stored in the streams can be accessed using programming libraries, today the NATS project
support Go, Java, JavaScript with .Net and C being in progress.  Ruby and Python to follow soon.

While the above list is what we have in mind medium term for Choria and JetStream, the Choria
Broker is a fully fledged JetStream instance and anything mentioned in the [NATS documentation](https://docs.nats.io/jetstream) 
is possible with Choria Streams.
