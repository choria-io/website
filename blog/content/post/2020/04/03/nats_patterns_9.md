---
title: "NATS Messaging - Part 9"
date: 2020-04-03T08:00:00+01:00
tags: ["nats", "development", "architecture", "jetstream"]
draft: false
---

[Yesterday](https://choria.io/blog/post/2020/04/02/nats_patterns_8/) we made some quite minor changes to our app and got it to use JetStream on both the Producer and Consumer side. These changes solved several problems for us, like being able to restart Consumers without losing any messages.

The last remaining issue was around handling messages that fail to be delivered.  Imagine the case where our disk is full on the Consumer, wouldn't it be great if we can somehow communicate our inability to handle messages to the network and have it retry later?

That's the role of Acknowledgements and JetStream supports several modes. Today we'll look at those.

<!--more-->
## Basic per-message Acknowledgements

On the basic level, what would probably be sufficient for our blog series is to Acknowledge every logline. This approach fails at large scale, but as we're just toying around and exploring approaches, this is perfectly fine.

Before getting into the detail I'd like to show while there are much to keep in mind, Acknowledgements can be very easy by adding them to our Log Consumer.

The Consumer has a loop like this today:

```go
for {
        select {
        case m := <-lines:
            err = handleLine(directory, output, m)
            if err != nil {
                log.Printf("Could not handle line %q: %s", string(m.Data), err)
            }

        case <-ctx.Done():
            // not shown
        }
    }
```

So we have a good place to know when message processing failed inside our case statement.  At the basic per-message case, we would change this to:

```go
for {
        select {
        case m := <-lines:
            err = handleLine(directory, output, m)
            if err != nil {
                log.Printf("Could not handle line %q: %s", string(m.Data), err)
                continue
            }

            m.Respond(nil)
        case <-ctx.Done():
            // not shown
        }
    }
```

By adding these 2 lines, we will Ack only successfully handled messages, ones that fail are retried later.

## Acknowledgement Modes

The devil is in the details though, so let's take a quick look at what JetStream supports. 

In a high-performance work queue, you have to be able to give up on a message and move to the next one. And your Acknowledgements must be optimised for lowest possible network round trips.

JetStream gives us several ways to optimise this often very costly part of Message Orientated architectures.

### No Acks

So far this is what we've been doing, when we configured the Consumers, we explicitly disabled Acknowledgements and just YOLO'd our data to disk, this is perfectly fine when just casually observing systems or with data that gets re-published very frequently.

### Explicit Acks

You might choose, as above, to Acknowledge every single message. This approach is great for Work Queues and the like where you are not looking at receiving thousands of messages a second. A message that has not been Acknowledged within a configured time gets redelivered later.

### _All_ Acks

This mode is suited to high throughput applications. You'd keep track of messages sequences and if you've successfully handled 100 messages, you can Ack just the 100th message. It will assume you have handled 1-99 correctly. This is a significant performance improvement and overhead reducer.

Logs can be slow at times, so you probably want to have an additional timer that checks regularly and does an Acknowledgement. Say you Ack only every 100 messages which at busy times is every 10 seconds. But when not much is happening, it could be hours to receive 100 messages. So the timer would check every minute and Ack regardless.

## Types of Acknowledgement

JetStream supports a few types of Acknowledgement to assist with controlling how and when retries happen.

### Ack

This mode is what we saw before, `msg.Respond(nil)` or `msg.Respond(0)`, and the server knows the message was successfully processed. Very simple and good for most use cases

### Nak

You might receive a message that you are not interested in or would rather send back to let others handle it. Typically you would have to wait for the redelivery period - often many minutes - for the Ack to timeout. In that time you might not be given a new message.  By sending a Nak, you tell JetStream you are not going to handle this message and to move on to the next one. This Ack is done via `msg.Respond([]byte("-NAK))`.

### Progress

The opposite of a Nak, here you might have 30 seconds to process a message, but as you get up to 25 seconds, you might still be retrying to access a remote service.  You can then do `msg.Respond([]byte("+WPI+))` and get another 30 seconds to handle the message.

### Next

This one is probably quite niche but can give you a noticeable boost under very high throughput scenarios. You can send the Acknowledgement and in the same packet ask JetStream to send you the next message without having to do another Pull operation.

```go
// wait for messages on q
ib := nc.NewRespInbox()
q := make(chan *nats.Msg)
nc.ChanSubscribe(ib, q)

for m := range q {
        // process the message

        // publish to the message reply target a +NXT acknowledgement asking
        // for a message on ib which will be placed in q
        nc.PublishRequest(m.Reply, ib, []byte("+NXT"))
}
``` 

This mode applies only to Pull Consumers, and it means you can Acknowledge+Pull in one packet rather than 2, on a busy system this is a significant optimisation.

## Redelivery

We mentioned earlier that Redelivery is based on a timeout. When you add the Consumer, you tell it to allow 30 seconds for a message to be processed using `--wait=30s`.  What you can also specify is how many Redelivery attempts the system should make; this is important because you might have a corrupt message that crashes all your consumers.  This message - often called a poison pill - can affect a DOS against everything.

Setting a Maximum Delivery limit of 10 lets these messages expire and not crash your system forever.  Maximum Delivery is set using `--max-deliver=10` when adding a Consumer.

The question is then what to do with these messages and how to debug them. In a traditional middleware, there is typically something called a Dead Letter Queue that holds the actual expired messages. This DLQ does not make sense in a Stream since the message might be consumed by 100s of Consumers and only 1 of them might be failing. We cannot remove the message, and we do not want to duplicate the message.

JetStream publishes various advisories about its lifecycle, and you can view these using `nats events`, this is how it looks when a message exceeds it's delivery attempts:

```nohighlight
$ nats events
Listening for Advisories on $JS.EVENT.ADVISORY.>
[20:36:28] [I7S817vOvH9NfO2MYJV8pE] Delivery Attempts Exceeded
         Consumer: WQ > WORK
  Stream Sequence: 5
       Deliveries: 2
```

Under the covers this is a JSON message published to `$JS.EVENT.ADVISORY.MAX_DELIVERIES.<stream name>.<consumer name>`:

```json
{
  "schema": "io.nats.jetstream.advisory.v1.max_deliver",
  "id": "I7S817vOvH9NfO2MYJV8lD",
  "timestamp": "2020-04-02T20:32:18.898715Z",
  "stream": "WQ",
  "consumer": "WORK",
  "stream_seq": 4,
  "deliveries": 2
}
``` 

This message is effectively a pointer to the message in the source Stream. I would suggest you always set up a stream that consumes `$JS.EVENT.>` and keep those for a few hours. Using that Stream, you can always get a view of what is going on in your account.

We can view the message in the Source Stream:

```nohighlight
$ nats stream get WQ 5
Item: WQ#5 received 2020-04-02 22:36:16.603899 +0200 CEST on Subject workq.jobs

{"job":"hello world"}
```

In this way, we support the traditional DLQ pattern while keeping the Streams intact and not duplicating the data.

## Conclusion

Message Acknowledgements is an essential and complicated topic, knowing which to choose for your case depends on many factors.

By applying these patterns to our log shipper, we have achieved all the goals that we set out to solve.

 * It can handle a very large number of log senders by being horizontally and vertically scalable
 * Ordering is maintained while being horizontally scalable
 * It's reliable, and we do not lose messages when disks go full

There is one last thing I'd like to explore, and that is a matter of overall System safety.

When thousands of nodes are publishing messages, the network is always under quite a lot of pressure, and it's easy for a system event to overwhelm your network.  

We might want to have a Big Red Button we can press to Pause message production for some time to give us a way to reduce the load on networks or transit.  I'll look at patterns for this next week.
