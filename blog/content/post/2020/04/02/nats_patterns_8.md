---
title: "NATS Messaging - Part 8"
date: 2020-04-02T08:00:00+01:00
tags: ["nats", "development", "architecture", "jetstream"]
draft: false
---

In our previous post, we dived a bit into JetStream API, and how to interact with it, many people would not need to know this all to get going. The CLI or Terraform management approaches would be perfectly fine. And today we'll use the CLI rather than the API.

In this post, we're back on our codebase, and we'll see how we might need to change the tools to support JetStream well. To be honest, I could have made some better decisions early on about the shipper design, but that gives us more opportunity to see how some apps might need to adapt.

<!--more-->
## Stream Design

In our previous posts, we saw messages published to subjects like `logs.p1.web1.example.net` and the Consumers would subscribe directly to those subjects. Things change a bit in JetStream, so we need to decide our approach.

 * We can put everything in one Stream
 * We can put each partition in a unique stream
 
A partition per Stream is an excellent choice for very large deploys; once JetStream has clustering, it might make for a better strategy with better scalability. If you expect huge volumes of data, a single Partition can limit your resiliency

For example, it could be that if you choose to keep 1TB of logs in the Stream that if one partition reader is down the volume received by other partitions could quickly expire the unconsumed partition. So separate partitions can give you much better control over the limits. You'll have to decide if this is something you need.

The design choices we made to bake the FQDN into the subject limits us a bit, so for now we have to use a Stream per partition. Later we might place the FQDN in the payload which would give us more options - or perhaps JetStream GA will work around the limitation we'll encounter below. Either way, we'll use a Stream per Partition here.

We can use a Stream Template to manage these per partition Streams. We'd create a template called `LOGS_T` that in the template listens on `logs.p.>`, this means when a new message arrives on `logs.p.1.web1.example.net` a new Stream will automatically be created for that partition. Stream Templates have some gotchas - they cannot be edited - but could be a good option. 

The other option is to pre-create the streams, which is what we'll do for 10 partitions: 

```nohighlight
$ for i in $(seq 0 9)
do
    nats str add "LOGS_${i}" --subjects "logs.p${i}.>" \
                    --storage file \
                    --retention limits \
                    --max-msgs=-1 \
                    --max-bytes=-1 \
                    --max-age 1d \
                    --max-msg-size=-1
done
```

```nohighlight
$ nats str ls
Streams:

        LOGS_0
        LOGS_1
        LOGS_2
        LOGS_3
        LOGS_4
        LOGS_5
        LOGS_6
        LOGS_7
        LOGS_8
        LOGS_9
```

## Stream Consumer Design

Consumers cannot filter streams by wildcards, we cannot filter on `logs.p1.>` if it was not for this limitation we could stick all the logs in a single Stream. Hopefully, JetStream gets this feature before JetStream goes GA. For now, we'll have to create a Consumer per Stream and later subscribe to those.

Push Consumers is the easiest to get going, and we set them to publish their data to `out.logs.p1` (no wildcard or FQDN, see later for how we access the wildcard).

```nohighlight
$ for i in $(seq 0 9)
do
    nats consumer add "LOGS_${i}" TAIL \
        --target "out.logs.p${i}" \
        --deliver all \
        --replay instant \
        --ack none \
        --filter ''
done
```

```nohighlight
$ nats consumer info LOGS_1 TAIL
Information for Consumer LOGS_1 > TAIL

Configuration:

        Durable Name: TAIL
    Delivery Subject: out.logs.p1
         Deliver All: true
          Ack Policy: none
       Replay Policy: instant

State:

  Last Delivered Message: Consumer sequence: 0 Stream sequence: 0
    Acknowledgment floor: Consumer sequence: 0 Stream sequence: 0
        Pending Messages: 0
    Redelivered Messages: 0
```

With this done and starting our Producer on a few nodes unmodified, we should see messages showing up in some of the Partitions. Also, note that each Partition has 1 Consumer defined.

```nohighlight
$ nats str report -n
Obtaining Stream stats

+--------+-----------+----------+--------+---------+----------+
| Stream | Consumers | Messages | Bytes  | Storage | Template |
+--------+-----------+----------+--------+---------+----------+
| LOGS_0 | 1         | 400      | 82 KiB | File    |          |
| LOGS_1 | 1         | 0        | 0 B    | File    |          |
| LOGS_2 | 1         | 200      | 60 KiB | File    |          |
| LOGS_3 | 1         | 0        | 0 B    | File    |          |
| LOGS_4 | 1         | 368      | 72 KiB | File    |          |
| LOGS_5 | 1         | 0        | 0 B    | File    |          |
| LOGS_6 | 1         | 0        | 0 B    | File    |          |
| LOGS_7 | 1         | 0        | 0 B    | File    |          |
| LOGS_8 | 1         | 0        | 0 B    | File    |          |
| LOGS_9 | 1         | 0        | 0 B    | File    |          |
+--------+-----------+----------+--------+---------+----------+
``` 

If we did subscribe to one of the partitions with data we'd see our logs.

```nohighlight
$ nats sub out.logs.p4
12:56:58 [#367] Received JetStream message: consumer: LOGS_4 > TAIL / subject: logs.p4.str1.example.net / delivered: 1 / consumer seq: 367 / stream seq: 367 / ack: false

Apr  2 12:26:43 grit syslogd[100]: ASL Sender Statistics
```

Note that while we subscribed to `out.logs.p4` the _subject_ that the message is reporting is `logs.p4.str1.example.net`, thus JetStream remembers and exposes to us that original subject complete with our additional information in the subject even though we subscribed to the delivery target of `out.logs.p4` - it preserves the original message metadata that we require.

It appears that it should be possible to subscribe to `out.logs.>` to get all the messages from all the Streams, but this is not supported - JetStream requires you to subscribe to the specific delivery target you want.

## Consuming the data from the Stream in Go

At this point, all our data is in the Streams without any modifications to the Producer. 

In our Consumer, we need to make 2 tiny changes to get the necessary log consumption to work:

 * We have to subscribe to a new per partition subject
 * We have to parse the original prefix
 
Thus far, we used the same data for these steps, so let's add a variable:

```go
	streamSource := os.Getenv("SHIPPER_STREAM_TARGET")
	if streamSource == "" {
		log.Fatalf("Please set a JetStream target subject using SHIPPER_STREAM_TARGET")
	}

	// lines removed

	// start consumers for every partition until ctx interrupt
	for _, partition := range partitions() {
		err := consumer(ctx, wg, streamSource, partition, directory, output)
		if err != nil {
			log.Fatalf("Consuming messages failed: %s", err)
		}
	}
```

With this the parsing out of hostnames will use the existing code and we will use the `SHIPPER_STREAM_TARGET` to construct our subscriptions.  Lets run the consumer:

```nohighlight
SHIPPER_OUTPUT=logfile NATS_URL=localhost:4222 SHIPPER_READ_PARTITIONS="0,1,2,3,4" SHIPPER_DIRECTORY=/tmp/shipper SHIPPER_SUBJECT=logs SHIPPER_STREAM_TARGET=out.logs ./consumer
2020/04/02 13:10:20 Waiting for messages on subject out.logs.p0 @ nats://localhost:4222
2020/04/02 13:10:20 Creating new log "/tmp/shipper/web1.example.net/logfile-%Y%m%d%H%M"
2020/04/02 13:10:20 Waiting for messages on subject out.logs.p1 @ nats://localhost:4222
2020/04/02 13:10:20 Waiting for messages on subject out.logs.p2 @ nats://localhost:4222
2020/04/02 13:10:20 Waiting for messages on subject out.logs.p3 @ nats://localhost:4222
2020/04/02 13:10:20 Creating new log "/tmp/shipper/web3.example.net/logfile-%Y%m%d%H%M"
2020/04/02 13:10:20 Waiting for messages on subject out.logs.p4 @ nats://localhost:4222
2020/04/02 13:10:20 Creating new log "/tmp/shipper/db1.example.net/logfile-%Y%m%d%H%M"
2020/04/02 13:10:20 Creating new log "/tmp/shipper/web2.example.net/logfile-%Y%m%d%H%M"
2020/04/02 13:10:20 Creating new log "/tmp/shipper/web4.example.net/logfile-%Y%m%d%H%M"
2020/04/02 13:10:20 Creating new log "/tmp/shipper/str1.example.net/logfile-%Y%m%d%H%M"
```

It subscribed to the Consumer targets and received the hostnames correctly, everything is fine.

## Conclusion

In the end, despite not making the best choices when I set out designing this getting a persistence feature was not too bad.

The main missing feature here is the ability to acknowledge messages, so failed messages that we could not consume, perhaps due to failed disks, can be redelivered later.

In our next post, we'll cover Acknowledgement models and add in support for that in our code as well as the way JetStream handles the traditional Dead Letter Queue pattern.

As before the full code and all modifications made for this post can be browsed at [GitHub @ripienaar fshipper tag post5](https://github.com/ripienaar/fshipper/tree/post8) 

