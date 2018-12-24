+++
title = "NATS Streaming"
weight = 200
+++

[NATS Streaming Server](https://github.com/nats-io/nats-streaming-server) is a easy to deploy and scale Streaming Server from the same people who make the NATS technology that the Choria Network Broker is built on.

The Choria NATS Streaming Adapter is hosted in the Choria Broker and is configured using the `choria::broker` class.

## Configuration

Given a NATS Streaming server setup with a cluster id of `prod_1`, the following will receive Registration data and re-publish it into a NATS Streaming Topic `node_data`.

```puppet
class{"choria::broker":
  adapters => {
    "node_data" => {
      "stream"  => {
        "type"      => "natsstream",
        "clusterid" => "prod_1",
        "topic"     => "node_data",
        "workers"   => 10,
        "servers"   => [
          "stan1.example.net:4222",
          "stan2.example.net:4222",
          "stan3.example.net:4222"
        ]
      },
      "ingest" => {
        "topic"    => "mcollective.broadcast.agent.discovery",
        "protocol" => "request",
        "workers"  => 10
      }
    }
  }
}
```

Many adapters can be hosted in a single Choria Broker.

## Data Formats

MCollective have traditionally supported publishing Registration messages into the broadcast network of MCollective. While this was useful it was very hard to process as the data tended to be highly concurrent and quite lossy.

The data that is received on the Choria side is kept as is and republished to the NATS Stream in the following format:

```json
{
  "data": "...the data received exactly as it was received....",
  "sender": "web1.ldn.example.net",
  "time": "2018-03-03T14:49:17Z"
}
```

## Choria Stream Replicator

The data is created to be specifically compatible with the [Choria Stream Replicator](https://github.com/choria-io/stream-replicator), using this tool combined with NATS Streaming and this Adapter you can create a metadata processing pipeline of tremendous scale capable of spanning the globe.

You'll have the ability to detect down agents quickly and using the Stream Replicators data inspection features be able to build a large tiered data pipeline that can span the globe.