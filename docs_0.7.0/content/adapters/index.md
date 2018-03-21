+++
title = "Data Adapters"
icon = "<b>7. </b>"
+++

Data Adapters is a Choria only technology that exists to convert data from the highly concurrent broadcast medium of Choria into formats more suitable to processing using technologies like Stream Processing.

![Adapters Overview](../adapters-overview.png)

The use cases for this include:

 * Receiving Registration data from large amounts of nodes and processing out of band
 * Receiving telemetry such as temperature and humidity from IoT devices that embed Choria Server into their systems
 * Creating secure setups where requestors will not be able to view replies - they go into something like Elastic Search
 * Scaling asynchronous services by storing replies to requests in a less real time medium

{{% notice warning %}}
At present only NATS Streaming is supported, this feature is under active development
{{% /notice %}}

Data adapters are hosted inside the Choria Brokers much the same way that Network Brokers and Federation Brokers can live in the same binary.
