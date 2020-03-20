---
title: "NATS Messaging - Part 4"
date: 2020-03-13T09:00:00+01:00
tags: ["nats", "development", "architecture"]
draft: false
---

Previously we used the `nats` utility to explore the various patterns of messaging in NATS, today we'll write a bit of code, and in the following few posts, we'll expand this code to show how to scale it up and make it resilient.

We'll write a tool that tails a log file and publish it over NATS to a receiver that writes it to a file. The intent is that several nodes use this log file Publisher and a central node Consume and save the data to a file per Publisher with log rotation in the central node.

I should be clear that there are already solutions to this problem, I am not saying you should solve this problem by writing your own.  It's a good learning experience, though because it's quite challenging to do right in a reliable and scalable manner.

 * Producers can be all over the world in many locations
 * It's challenging to scale problem as you do not always control the rate of production and have an inherent chokepoint on the central receiver
 * You do not want to lose any logs, so we probably need persistence
 * Ordering matters, there's no point in getting your logs in random order
 * Scaling consumers horizontally while having order guarantees is difficult, additionally you need to control who writes to log files
 * Running a distributed network with all the firewall implications in enterprises is very hard

So we'll look if we can build a log forwarder and receiver that meet these criteria, in the process we'll explore the previous sections in depth.

We'll use Go for this but the NATS ecosystem support over 30 languages today, you're spoiled for choice.

<!--more-->

We'll start with a simple log tail and publishing tool. Tailing logs is quite a difficult problem, especially when you consider log rotations, restarts and more.  For a system like this, you'll need to remember the position you got to and continue on restart. We'll punt on some of this a bit and use an Open Source [Tail Library](https://godoc.org/github.com/hpcloud/tail) to build this; it does not have automatic continuation after restart so we'll skip that feature. But that is ok as the focus is the NATS architecture.

![](/blog/mom/log-pipeline-overview.png)

On the NATS side, we'll publish to a configurable subject, we'll support reconnecting on disconnections and in general try to be resilient - but for sure more is needed than we'd implement. 

We'll start with a basic feature set by creating a system where 1 Publisher publishes on a subject, and 1 Consumer reads it, this is far from our goal, but it's step one in an iterative process.

## Overview

To get started we'll build something suitable to 1 publisher and 1 consumer, it's a start and gives us something to iterate around. At this point, the code listings are long because there's a lot of boilerplate around signal handling and just generally making a program.

The programs are configurable using environment variables, here’s a list of all we’ll support:

|Environment|Description|Required|Example|
|-----------|-----------|--------|-------|
|NATS_URL   |Servers to connect to|yes|`nats://n1.my.new:4222,nats://n2.my.net:4222`|
|NATS_CREDS |NATS 2.0 credentials to authenticate with||`/etc/fshipper/nats.creds`|
|NATS_CERTIFICATE|Public certificate to use for TLS||`/etc/fshipper/cert.pem`|
|NATS_KEY|Private key to use for TLS||`/etc/fshipper/key.pem`|
|NATS_CA|Certificate Authority chain for TLS||`/etc/fshipper/ca.pem`|
|SHIPPER_SUBJECT|The NATS subject to publish to|yes|`shipper`|
|SHIPPER_FILE|The file to publish/consume|yes|`/var/log/system.log`|
|SHIPPER_OUTPUT|The file to write to|yes|`/var/log/remote/system.log` or `/var/log/remote/system.log.%Y%m%d`|

Here the `SHIPPER_OUTPUT` takes a pattern and it will write log lines to that, it will rotate daily and keep a weeks worth (something to make configurable ideally).

Here is a quick tour of the main parts of the programs we'll write.  The final code is more complicated than this, but these are the essential parts.

We connect to the middleware in a helper function used by both the Consumer and Producer:

{{< gist ripienaar bdc3d7be530f563686c5480bfb084e3f >}}

Establishing the connection can be complicated if you want to handle certificates, credentials and more. Above is the basics that set up automatic reconnects and some logging callbacks, see the full code listing for the callbacks and how we handle authentication and TLS.

On the publishing side, once connected, we open the file and publish every line:

{{<gist ripienaar adb4889907a665ce3812cee51bbca75d >}}

The `tail` package does the hard work, we could store our most recent location in a state file and make sure we continue where we left off later to avoid republishing the entire file on every start - but the focus here is the NATS parts, so we stick to basic tail handling.

And finally, on the consumer we use a Go channel and `ChanSubscribe()` to put the messages into our local buffer which we then write to file forever.

{{<gist ripienaar 4c50931478999dc6f07e53d8c557e0e7 >}}

Again we outsource the hard work of rotating logs and deleting old logs to a package, we could make more of this configurable, but we're staying focused on the NATS bits here.
 
Below you'll see the whole program expanded with reading config from the environment and more. The source code seen here can be browsed directly on [GitHub @ripienaar fshipper tag post4](https://github.com/ripienaar/fshipper/tree/post4).

## Getting Connected

We use the [nats.go](https://github.com/nats-io/nats.go/) package to connect to NATS, so far we've shown this as a 1 liner, but you'd want to do a bit more, you'll need to support configurable server URLs, optional TLS and optional authentication credentials.

{{< gist ripienaar b230ccca36875647550fd7b97a0f4dec >}}

Calling `NewConnection()` will set up a new NATS connection that will forever reconnect and gets configured from the environment.

This helper is used for all connections in both the Producer and the Consumer, so it's worth making it robust and configurable.

We added 1 other little utility there related to `^C` handling.

## Producing log lines

Let's look at the producer; this is a stand-alone application compiled into a single binary.

<br>
{{< gist ripienaar ae20c4c72c87a83bd75a3fdf805ca929 >}}

That's a basic starting block for shipping the file, every time this starts it reads the file and sends it's entire contents and then follows it forever - even through log rotations. It's not perfect, but it's a start, we don't want to get lost in details of file tailing here (remember, use an off the shelve tool for real).

You can test this by running the binary using `SHIPPER_FILE=/var/log/system.log SHIPPER_SUBJECT="shipper" NATS_URL="localhost:4222" ./producer` and using `nats sub shipper` to verify it works.

## Consuming log lines

Let's create a Consumer. We'll listen on a subject and write everything we receive directly to a file; the file rotates daily. The consumer is a bit more complicated there's a lot of setup and `^C` handling and so forth.

{{< gist ripienaar 2c63f4c92c7e4d4b02a77dff2621a513 >}}

Running this like `SHIPPER_SUBJECT=shipper SHIPPER_OUTPUT=/tmp/logfile NATS_URL=localhost:4222 ./consumer` will create `/tmp/logfile-202003180000` that rotates daily.

## Conclusion

So this is a very basic file Tail tool and a Consumer, it has several shortcomings:

 * 1 Producer/Consumer pair per host is required to split the lines by host
 * If you tried to scale the Consumer up, you start getting log lines out of order, and the file might get corrupt

So basically this is far from fit for purpose, but it does show we can publish data and receive it in a reasonably robust manner, and we can get our data to a central point.

There is though one nice side effect here, we mentioned how it's suitable for a 1:1 Producer to Consumer meaning your files is stored in one place only. What isn't clear is that you could start the same Consumer on another node and it will get the whole log too automatically creating redundancy in storage without any file system level syncs. A great example of the freebie benefits from using middleware for this kind of system.

Next posts we'll improve this to be better suited to our stated problem.
