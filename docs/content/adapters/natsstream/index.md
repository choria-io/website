+++
title = "NATS Streaming"
weight = 100
+++

[NATS Streaming Server](https://github.com/nats-io/nats-streaming-server) is a easy to deploy and scale Streaming Server from the same people who make the NATS technology that the Choria Network Broker is built on.

The Choria NATS Streaming Adapter is hosted in the Choria Broker and is configured using the `choria::broker` class.

Given a NATS Streaming server setup with a cluster id of `prod_1`, the following will receive Registration data and re-publish it into a NATS Streaming Topic `node_data`.

```puppet
class{"choria::broker":
  adapters => {
    "node_data" => {
      "stream" => {
        "type" => "natsstream",
        "clusterid" => "prod_1",
        "topic" => "node_data",
        "workers" => 10,
        "servers" => [
          "stan1.example.net:4222",
          "stan2.example.net:4222",
          "stan3.example.net:4222"
        ]
      },
      "ingest" => {
        "topic" => "mcollective.broadcast.agent.discovery",
        "protocol" => "request",
        "workers" => 10
      }
    }
  }
}
```

Many adapters can be hosted in a single Choria Broker.