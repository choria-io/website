+++
title = "JetStream"
weight = 100
+++

NATS JetStream is a easy to deploy and scale Streaming Server from the same people who make the NATS technology that the Choria Network Broker is built on.  JetStream will eventually replace NATS Streaming as a closer integrated streaming server and it's conceivable we will support JetStream natively in the Choria Broker.

{{% notice warning %}}
JetStream and our support for it is currently in Technology Preview, things will change, things will break, data might get lost. This is supported in Choria Server version 0.13.0 and newer.
{{% /notice %}}

The Choria JetStream Adapter is hosted in the Choria Broker and is configured using the `choria::broker` class.

## Configuration

Given a JetStream server, the following will receive Registration data and re-publish it into a JetStream Topic `choria.node_metadata.%s` where the `%s` will be replaced with the sender node id.

```puppet
class{"choria::broker":
  adapters => {
    "node_data" => {
      "stream"  => {
        "type"      => "jetstream",
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

## JetStream Setup

You have to create a `message set` in your JetStream that matches the above, here we create one that keeps messages for a week:

```nohighlight
$ nats stream add CHORIA_REGISTRATION
...
```

## Data Formats

MCollective has traditionally supported publishing Registration messages into the broadcast network of MCollective. While this was useful it was very hard to process as the data tended to be highly concurrent and quite lossy.

The data that is received on the Choria side is kept as is and republished to JetStream in the following format:

```json
{
  "protocol": "choria:adapters:jetstream:output:1",
  "data": "...the data received exactly as it was received....",
  "sender": "web1.ldn.example.net",
  "time": "2018-03-03T14:49:17Z"
}
```

You can find a JSON Schema for this in our [Schemas Repository](https://choria.io/schemas/choria/adapters/jetstream/v1/output.json).
