+++
title = "JetStream"
weight = 200
+++

NATS JetStream is a easy to deploy and scale Streaming Server from the same people who make the NATS technology that the Choria Network Broker is built on.  JetStream will eventually replace NATS Streaming as a closer integrated streaming server and it's conceivable we will support JetStream natively in the Choria Broker.

{{% notice warning %}}
JetStream and our support for it is currently in Technology Preview, things will change, things will break, data might get lost. This is supported in Choria Server version 0.13.0 and newer.
{{% /notice %}}

The Choria JetStream Adapter is hosted in the Choria Broker and is configured using the `choria::broker` class.

## Configuration

Given a JetStream server, the following will receive Registration data and re-publish it into a NATS Streaming Topic `node_data`.

```puppet
class{"choria::broker":
  adapters => {
    "node_data" => {
      "stream"  => {
        "type"      => "jetstream",
        "topic"     => "node_data",
        "workers"   => 10,
        "servers"   => [
          "js1.example.net:4222",
        ]
      },
      "ingest" => {
        "topic"    => "choria.node_metadata",
        "protocol" => "request",
        "workers"  => 10
      }
    }
  }
}
```

Many adapters can be hosted in a single Choria Broker.

## JetStream Setup

You have to create a `message set` in your JetStream that match the above, here we create one that keeps messages for a week:

```nohighlight
$ jsm add
Enter the following information
Name: choria_discovery
Subjects: choria.node_metadata
Limits (msgs, bytes, age): -1 -1 1w
Storage: file
Received response of "+OK"
```

After a while you'll start seeing messages if you configured your nodes to publish their metadata to `choria.node_metadata`:

```nohighlight
$ jsm info choria_discovery
Messages: 2
Bytes:    12 kB
FirstSeq: 1
LastSeq:  2
```

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