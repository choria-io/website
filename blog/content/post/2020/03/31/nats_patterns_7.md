---
title: "NATS Messaging - Part 7"
date: 2020-03-31T08:00:00+01:00
tags: ["nats", "development", "architecture"]
draft: false
---

[Yesterday](https://choria.io/blog/post/2020/03/30/nats_patterns_6/) we did a quick intro to JetStream, before we jump in and write some code we have to talk a bit about how to configure it via its API and how it relates to core NATS.

NATS Streaming Server, while built using the NATS broker for communication, is, in fact, a different protocol altogether.  It's relation to NATS is more like that of HTTP to TCP.  It uses NATS transport, but its protocol is entirely custom and uses Protobuf messages. This design presented several challenges to users - authentication and authorization specifically were quite challenging to integrate with NATS.

NATS 2.0 brought a significant rework of the Authentication and Authorization in NATS and integrating the new world with NATS Streaming Server would have been too disruptive. Further NATS 2.0 is Multi-Tenant which NATS Streaming Server couldn't be without a massive rework.

So JetStream was started to be a much more natural fit in the NATS ecosystem, indeed, as you saw Yesterday the log shipper Producer did not need a single line of code change to gain persistence via JetStream. Additionally, it is a comfortable fit in the Multi-tenant land of NATS 2.0. All the communication uses plain NATS protocol, and some JSON payloads in its management API.

<!--more-->
## API Overview

Interacting with JetStream uses the patterns you already learned in this series. Using the API to create Streams its a Request/Reply pattern and when Consuming or Producing messages in many cases, its the Pub/Sub pattern. 

The JetStream API communicates via standard Subjects, below a partial list:

|Subject|Description|
|-------|-----------|
|`$JS.ENABLED`|Determines if JetStream is enabled|
|`$JS.INFO`|General account information|
|`$JS.STREAM.LIST`|List all streams|
|`$JS.STREAM.<stream>.CREATE`|Creates a Stream|
|`$JS.STREAM.<stream>.UPDATE`|Updates a Stream configuration|
|`$JS.STREAM.<stream>.INFO`|Retrieve Stream information|
|`$JS.STREAM.<stream>.DELETE`|Delete a Stream|
|`$JS.STREAM.<stream>.PURGE`|Delete all messages in a Stream|
|`$JS.STREAM.<stream>.MSG.DELETE`|GDPR compliant deletion of a single message from a Stream|
|`$JS.STREAM.<stream>.CONSUMERS`|List Consumers for a Stream|
|`$JS.STREAM.<stream>.CONSUMER.<consumer>.CREATE`|Create a Consumer|
|`$JS.STREAM.<stream>.CONSUMER.<consumer>.INFO`|Retrieve Consumer information|
|`$JS.STREAM.<stream>.CONSUMER.<consumer>.DELETE`|Delete a Consume|
|`$JS.STREAM.<stream>.EPHEMERAL.CONSUMER.CREATE`|Create a temporary Consumer|

You can see to a large extend this resembles a REST API design with standard CRUD style actions. These Subject name choices make it easy to construct Authorization rules to give users fine-grained access to how they can maintain JetStream.

You're welcome to make the Requests in code however you like, for Go, there is a Work In Progress package called [jsm.go](https://github.com/nats-io/jsm.go) that lets you perform all management functions that the `nats` utility allows.

## Interacting with the API

As I mentioned interacting with the API is as easy as using the patterns you already learned.

Check if a specific account has JetStream enabled:

```nohighlight
$ nats req '$JS.ENABLED' '.'
11:35:11 Sending request on [$JS.ENABLED]
11:35:11 Received on [_INBOX.5883caztE1lVKyPRgaGfW4.fSU9BTtv]: '+OK'
```

Most JetStream APIs respond with `-ERR <reason>` on failure, endpoints like this that are boolean respond with `+OK` or `-ERR <reason>` or in some cases there would be no response.  Here for example, without JetStream you'll receive a timeout error as nothing is there to respond.

```nohighlight
$ nats req '$JS.ENABLED' '.'
11:36:58 Sending request on [$JS.ENABLED]
nats: error: nats: timeout, try --help
``` 

Other APIs respond with JSON, let's see our account limits:

```nohighlight
$ nats req '$JS.INFO' '.'
11:37:50 Received on [_INBOX.kEwkoIVeAzUpZHitEGaO8z.BhiZu4gH]: '{
  "memory": 0,
  "storage": 0,
  "streams": 1,
  "limits": {
    "max_memory": 1564062720,
    "max_storage": 1099511627776,
    "max_streams": -1,
    "max_consumers": -1
  }
}'
```

## Creating a Stream

Let's jump right in and look at a Stream configuration.  First, we re-create the `LOGS` Stream, so we have something there to work with:

```nohighlight
$ nats str add LOGS --subjects "logs.>" \
                    --storage file \
                    --retention limits \
                    --max-msgs=-1 \
                    --max-bytes=-1 \
                    --max-age 1d \
                    --max-msg-size=-1
```

You can also use the same utility to see the raw JSON responses:

```nohighlight
$ nats stream info LOGS -j
{
  "config": {
    "name": "LOGS",
    "subjects": [
      "logs.>"
    ],
    "retention": "limits",
    "max_consumers": -1,
    "max_msgs": -1,
    "max_bytes": -1,
    "max_age": 86400000000000,
    "max_msg_size": -1,
    "storage": "file",
    "num_replicas": 1
  },
  "state": {
    "messages": 0,
    "bytes": 0,
    "first_seq": 0,
    "last_seq": 0,
    "consumer_count": 0
  }
}
```

This command sent a Request to `$JS.STREAM.LOGS.INFO` and showed the response. The interesting bit here is the `config` key, that's the exact data you would send to create a stream from scratch. First, we remove the `LOGS` Stream:

```nohighlight
$ nats stream rm LOGS -f
$ nats stream info LOGS
nats: error: could not pick a Stream to operate on: no Streams are defined
```

With the JSON from the config key above in a file `LOGS.json` we can create it using a standard Request:

```nohighlight
$ cat LOGS.json
{
    "name": "LOGS",
    "subjects": [
      "logs.>"
    ],
    "retention": "limits",
    "max_consumers": -1,
    "max_msgs": -1,
    "max_bytes": -1,
    "max_age": 86400000000000,
    "max_msg_size": -1,
    "storage": "file",
    "num_replicas": 1
}
$ cat LOGS.json | nats req '$JS.STREAM.LOGS.CREATE'
11:54:05 Reading payload from STDIN
11:54:05 Sending request on [$JS.STREAM.LOGS.CREATE]
11:54:05 Received on [_INBOX.B1nlp66N0vy6RJRfMt2OzF.c7r3nbr4]: '+OK'
$ nats str info LOGS
Information for Stream LOGS

Configuration:

             Subjects: logs.>
     Acknowledgements: true
            Retention: File - Limits
             Replicas: 1
     Maximum Messages: unlimited
        Maximum Bytes: unlimited
          Maximum Age: 1d0h0m0s
 Maximum Message Size: unlimited
    Maximum Consumers: unlimited

State:

            Messages: 0
               Bytes: 0 B
            FirstSeq: 0
             LastSeq: 0
    Active Consumers: 0
```

### Creating Consumers

Consumers are roughly the same story; you create the consumer configuration as a JSON string and publish that to `$JS.STREAM.LOGS.CONSUMER.TAIL.CREATE` to create a `TAIL` Consumer within the `LOGS` Stream.

Grab an existing configuration and save it:

```nohighlight
$ nats consumer info LOGS TAIL -j> LOGS_TAIL.json
$ nats consumer rm LOGS TAIL -f
```

This configuration needs to a small edit, see below, you can use `nats consumer create --config LOGS_TAIL.json` to recreate it, but for completeness, I'll show the direct API route again.

```nohighlight
$ cat LOGS_TAIL.json
{
  "stream_name": "LOGS",
  "config": {
    "delivery_subject": "out.tail",
    "durable_name": "TAIL",
    "start_time": "0001-01-01T00:00:00Z",
    "deliver_all": true,
    "ack_policy": "none",
    "max_deliver": -1,
    "replay_policy": "instant"
  }
}
$ cat LOGS_TAIL.json|nats request '$JS.STREAM.LOGS.CONSUMER.TAIL.CREATE'
```

## Configuration Management

Ultimately these methods above are a bit tedious, but if you need a whole Stream and Consumer in code you can use the raw methods or the [jsm.go](https://github.com/nats-io/jsm.go) Package. 

Several options exist to perform automated Configuration Management that improves the experience and can add repeatability and configuration auditing.

We can edit the Stream from the CLI, first, we edit the `LOGS.json` file and set `"max_bytes": 10737418240,` to limit our maximum Logs archive to 10GB:

```nohighlight
$ nats stream edit LOGS --config LOGS.json
Differences (-old +new):
  server.StreamConfig{
        ... // 3 identical fields
        MaxConsumers: -1,
        MaxMsgs:      -1,
-       MaxBytes:     -1,
+       MaxBytes:     10737418240,
        MaxAge:       s"24h0m0s",
        MaxMsgSize:   -1,
        ... // 4 identical fields
  }
? Really edit Stream LOGS Yes
Stream LOGS was updated

Information for Stream LOGS

...
        Maximum Bytes: 10 GiB
...
```

This way you can store your Stream configs and apply them programmatically - perhaps during your CI runs.

You can also use Terraform with the [terraform-provider-jetstream](https://github.com/nats-io/terraform-provider-jetstream), the LOGS Stream and a TAIL Consumer would look like this:

```terraform
provider "jetstream" {
  servers = "localhost"
}

resource "jetstream_stream" "LOGS" {
  name     = "LOGS"
  subjects = ["logs.*"]
  storage  = "file"
  max_age  = 60 * 60 * 24 * 365
}

resource "jetstream_consumer" "LOGS_TAIL" {
  stream_id      = jetstream_stream.LOGS.id
  durable_name   = "TAIL"
  deliver_all    = true
}
```

## Conclusion

This post was a bit of whirlwind through the JetStream API. A lot is going on here; the main take away is that the API is a very familiar JSON based API using standard interactions. There is a [jsm.go](https://github.com/nats-io/jsm.go )Go package that can help you manage JetStream from your programs, hopefully with a subset of this in other languages later.

We saw the CLI tooling is a non-interactive CLI API and there is also a [Terraform provider](https://github.com/nats-io/terraform-provider-jetstream) with other Configuration Management systems support added based on community demand.

As before the [JetStream Technical Preview](https://github.com/nats-io/jetstream#readme) repository covers these areas in detail and should be your port of call if you want to dig into the detail of anything you saw here.

Tomorrow we'll get back to our log shipper and see how JetStream let us solve the problems we set out to solve.
