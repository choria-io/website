---
title: "NATS Messaging - Part 3"
date: 2020-03-12T09:00:00+01:00
tags: ["nats", "development", "architecture"]
draft: false
---

In our previous posts, we did a thorough introduction to messaging patterns and why you might want to use them, today lets get our hands dirty by setting up a NATS Server and using it to demonstrate these patterns.

## Setting up

Grab one of the zip archives from the [NATS Server Releases page](https://github.com/nats-io/nats-server/releases), inside you'll find `nats-server`, it's easy to run:

```nohighlight
$ ./nats-server -D
[26002] 2020/03/17 15:30:25.104115 [INF] Starting nats-server version 2.2.0-beta
[26002] 2020/03/17 15:30:25.104238 [DBG] Go build version go1.13.6
[26002] 2020/03/17 15:30:25.104247 [INF] Git commit [not set]
[26002] 2020/03/17 15:30:25.104631 [INF] Listening for client connections on 0.0.0.0:4222
[26002] 2020/03/17 15:30:25.104644 [INF] Server id is NA3J5WPQW4ELF6AJZW5G74KAFFUPQWRM5HMQJ5TBGRBH5RWL6ED4WAEL
[26002] 2020/03/17 15:30:25.104651 [INF] Server is ready
[26002] 2020/03/17 15:30:25.104671 [DBG] Get non local IPs for "0.0.0.0"
[26002] 2020/03/17 15:30:25.105421 [DBG]   ip=192.168.88.41
```

The `-D` tells it to log verbosely, at this point you'll see you have port `4222` open on `0.0.0.0`.

Also grab a copy of the `nats` CLI and place this in your path, this can be found in the [JetStream Releases page](https://github.com/nats-io/jetstream/releases), you can quickly test the connection:

```nohighlight
$ nats rtt
nats://localhost:4222:

   nats://127.0.0.1:4222: 251.289µs
       nats://[::1]:4222: 415.944µs
```

Above shows all you need to have a NATS Server running in development and production use is not much more complex, to be honest, once it's running this can happily serve over 50 000 Choria nodes depending on your hardware (a $40 Linode would do)

<!--more-->

## Basics

To do the following demos you will need a few shells running, I suggest using `tmux` but however you like will do. 

### Basic Pub/Sub

A refresher that Pub/Sub is the pattern where one Producer publish a message, and all consumers receive it, this is the most basic pattern in NATS.

![](/blog/mom/pub-sub.png)

<script id="asciicast-8V8PZvhZYnBI8e3xo2Xc3eBbC" src="https://asciinema.org/a/8V8PZvhZYnBI8e3xo2Xc3eBbC.js?autoplay=0&size=small" async></script>

Here, using the `nats` utility, we did a basic Subscribe on multiple consumers and Published a message to them all.  Things to note:

 * The Producer has no idea how many listeners there are, it has to do no additional work to scale to more consumers
 * The Consumers can scale and change their subscribe patterns and get only the messages that they specifically care for
 * We do not need to think in terms of addresses, we think about the type of message and conventions for where they live
 * Our Producer do not need to keep state of who its subsribers are like webhook based systems, this is all outsourced to the Middleware
 * The entire architecture is decoupled and non prescriptive

In Go this would be quite simple to do:

```go
// consumer
nc, _ := nats.Connect("localhost:4222")

nc.Subscribe("demo.hello", func(m *nats.Msg) {
    fmt.Prinf("Received a message: %s\n", string(m.Data))
})
```

```go
// producer
nc, _ := nats.Connect("localhost:4222")
nc.Publish("demo.hello", []byte{"hello world"})
```

### Horizontally scaled Streams

We know how to deliver a message to all consumers; let's see about making a queue group that allows us to share our workload in a cluster.

![](/blog/mom/queue-grp.png)

<script id="asciicast-Bycsu10BItgEMuwlv89aQX0vJ" src="https://asciinema.org/a/Bycsu10BItgEMuwlv89aQX0vJ.js?autoplay=0&size=small" async></script>

Here we demonstrated the creation of a queue group `grp1` but also how the queued mode and the normal Pub/Sub mode can co-habit, but also that the Message Producer requires no change from our previous demo.

In Go our producer would be the same, here is a queue group consumer:

```go
// consumer
nc, _ := nats.Connect("localhost:4222")

nc.QueueSubscribe("demo.hello", "grp1", func(m *nats.Msg) {
    fmt.Prinf("Received a message: %s\n", string(m.Data))
})
```

### Requests expecting a Reply

Next, we explore a service, still using the NATS CLI to set up a Highly Available and Load Shared service that responds to service requests, in less than 1 minute :-)

![](/blog/mom/weather-service.png)

<script id="asciicast-3ZGdZCz4AGJ0mW2IRxhOqV1Sf" src="https://asciinema.org/a/3ZGdZCz4AGJ0mW2IRxhOqV1Sf.js?autoplay=0&size=small" async></script>

That was really easy and the code is not much harder, first the Service:

```go
nc, _ := nats.Connect("localhost:4222")

nc.QueueSubscribe("demo.service", "grp1", func(m *nats.Msg) {
    m.Respond("hello world")
})
```

And now the Client:

```go
nc, _ := nats.Connect("localhost:4222")

// we wait up to 1 second for the reply after sending 'hello'
msg, _ := nc.Request("demo.service", []byte("hello"), time.Second)
```

That's all there is to it. 

This basic pattern allows you to achieve horizontal scalable, highly available services with automatic failover between regions in a network-distance aware way and as we'll see get observability for your service performance for free.

## Conclusion

Today we used the `nats` CLI tool to interact with a real running NATS Server and saw that sharing information between processes and setting up quite complicated patterns allowing for full HA, Failover and horizontally scalability was effortless.  

Next we'll look at more complicated scenarios and more code.
