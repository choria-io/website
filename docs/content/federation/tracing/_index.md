+++
title = "Packet Tracing"
weight = 300
+++

The packet flow through a Federated Choria Network can be traced using *mco federation trace dev1.example.net*.  It will publish a specially crafted message to the entire Federation that instructs it to trace the full path the packet takes through your network.  It will then show the result in a hopefully meaningful way.

![Federation Flow](../../federation_flow.png)

The network diagram above shows a Federated Collective and how a packet might flow across the network. When doing a trace you will see output like this:

![Federation Flow](../../federation_full_trace.png)

Take note of a few things here:

  * The message took an asymmetric route through the Federation Broker - Federation Broker Instances *choria1.ldn.example.net* and *choria2.ldn.example.net* were involved
  * Within the Federation Brokers are a number of micro services instances, individual service instances are shown - Prometheus stats would expose individual microservers as well
  * The *Federation Broker Instances* shown are only the ones that can be inferred from the discovered route since there is no central registry
  * The list of known *Federation Broker Clusters* is influenced by your configuration and the *CHORIA_FED_COLLECTIVE* variable

{{% notice tip %}}
This is unlike a traceroute, it can only show the result if the entire flow works and the reply is received.
{{% /notice %}}
