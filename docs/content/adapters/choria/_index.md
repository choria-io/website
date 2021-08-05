+++
title = "Choria Streams"
weight = 100
+++

NATS JetStream is a easy to deploy and scale Streaming Server from the same people who make the NATS technology that the Choria Network Broker is built on.  JetStream will eventually replace NATS Streaming as a closer integrated streaming server and it's conceivable we will support JetStream natively in the Choria Broker.

Choria embeds JetStream same as it does NATS, in choria JetStream is known as Choria Streams.

The Choria Streams Adapter is hosted in the Choria Broker and is configured using the `choria::broker` class.

## Configuration

Given a JetStream server, the following will receive Registration data and re-publish it into a JetStream Topic `choria.node_metadata.%s` where the `%s` will be replaced with the sender node id.

```puppet
class{"choria::broker":
  adapters => {
    "node_data" => {
      "stream"  => {
        "type"      => "choria_streams",
        "topic"     => "choria.node_metadata.%s",
        "workers"   => 10,
        "servers"   => [
          "js1.example.net:4222",
        ]
      },
      "ingest" => {
        "topic"    => "choria.discovery",
        "protocol" => "request",
        "workers"  => 10
      }
    }
  }
}
```

Many adapters can be hosted in a single Choria Broker.

## Choria Streams Setup

You have to create a `stream` in your Choria Streams instance that matches the above, here we create one that keeps messages for a week:

```nohighlight
$ nats stream add CHORIA_REGISTRATION
...
```

## Data Formats

MCollective has traditionally supported publishing Registration messages into the broadcast network of MCollective. While this was useful it was very hard to process as the data tended to be highly concurrent and quite lossy.

The data that is received on the Choria side is kept as is and republished to JetStream in the following format:

```json
{
  "protocol": "choria:adapters:choria_streams:output:1",
  "data": "...the data received exactly as it was received....",
  "sender": "web1.ldn.example.net",
  "time": "2018-03-03T14:49:17Z"
}
```

You can find a JSON Schema for this in our [Schemas Repository](https://choria.io/schemas/choria/adapters/choria_streams/v1/output.json).
