+++
title = "Packet Tracing"
weight = 300
+++

As of MCollective *2.10.3* you can trace the network route that a message takes through your Federation.

Once you invoke *mco federation trace dev1-1.ldn.example.net* it will publish a specially crafted message to the entire Federation that instructs it to trace the full path the packet takes through your network.  It will then show the result in a hopefully meaningful way.

![Federation Flow](../../federation_flow.png)

The network diagram above shows a Federated Collective and how a packet might flow across the network.  This is the pathological case where not one single part in the flow shares any infrastructure.  When doing a trace you will see output like this:

![Federation Flow](../../federation_full_trace.png)

Take note of a few things here:

  * The message took an asymmetric route through the Federation Broker - Federation Broker Instances *production:1* and *production:2* were involved
  * The *Federation Broker Instances* shown are only the ones that can be inferred from the discovered route since there is no central registry
  * The list of known *Federation Broker Clusters* is influenced by your configuration and the *CHORIA_FED_COLLECTIVE* variable
  * Every NATS hop is unique indicating that no 2 components were using the same NATS nodes in the clusters

The output will adjust in cases where there is shared infrastructure to make this clearer, here is the above flow if *production:2* Federation Broker Instance shared a NATS instance with the client and the node:

![Federation Flow](../../federation_full_trace_shared.png)

The trace tool also support unfederated tracing, below is a trace for a client that connects to the *london* Collective directly without using Federation, this graph too will contract and expand depending on how much of the infrastructure is shared and if the paths are symmetrical:

![Collective Flow](../../unfederated_full_trace.png)
