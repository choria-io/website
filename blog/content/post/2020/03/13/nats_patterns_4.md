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

![](/blog/mom/log-pipeline-overview.png)

<!--more-->

We'll start with a simple log tail and publishing tool. Tailing logs is quite a difficult problem, especially when you consider log rotations, restarts and more.  For a system like this, you'll need to remember the position you got to and continue on restart. We'll punt on some of this a bit and use an Open Source [Tail Library](https://godoc.org/github.com/hpcloud/tail) to build this; it does not have automatic continuation after restart so we'll skip that feature.

On the NATS side, we'll publish to a configurable subject, we'll support reconnecting on disconnections and in general try to be resilient - but for sure more is needed than we'd implement. 

We'll start with a basic feature set by creating a system where 1 Publisher publishes on a subject, and 1 Consumer reads it, this is far from our goal, but it's step one in an iterative process.

## Getting Connected

We use the [nats.go](https://github.com/nats-io/nats.go/) package to connect to NATS, so far we've shown this as a 1 liner, but you'd want to do a bit more, you'll need to support configurable server URLs, optional TLS and optional authentication credentials.

```go
// internal/util/util.go
package util

import (
	"fmt"
	"log"
	"os"
	"time"
    "os/signal"
    "syscall"
    "context"

	"github.com/nats-io/nats.go"
)
// NewConnection creates a new NATS connection configured from the Environment
func NewConnection() (nc *nats.Conn, err error) {
	servers := os.Getenv("NATS_URL")
	if servers == "" {
		return nil, fmt.Errorf("specify a server to connect to using NATS_URL")
	}

	opts := []nats.Option{
		nats.MaxReconnects(-1),
		nats.ErrorHandler(errorHandler),
		nats.ReconnectHandler(reconnectHandler),
		nats.DisconnectErrHandler(disconnectHandler),
	}

	if os.Getenv("NATS_CREDS") != "" {
		opts = append(opts, nats.UserCredentials(os.Getenv("NATS_CREDS")))
	}

	if os.Getenv("NATS_CERTIFICATE") != "" && os.Getenv("NATS_KEY") != "" {
		opts = append(opts, nats.ClientCert(os.Getenv("NATS_CERTIFICATE"), os.Getenv("NATS_KEY")))
	}

	if os.Getenv("NATS_CA") != "" {
		opts = append(opts, nats.RootCAs(os.Getenv("NATS_CA")))
	}

	// initial connections can error due to DNS lookups etc, just retry, eventually with backoff
	for {
		nc, err := nats.Connect(servers, opts...)
		if err == nil {
			return nc, nil
		}

		time.Sleep(500 * time.Millisecond)
	}
}

// SigHandler sets up interrupt signal handlers
func SigHandler() chan os.Signal {
	sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, syscall.SIGINT)

	return sigs
}

// called during errors subscriptions etc
func errorHandler(nc *nats.Conn, s *nats.Subscription, err error) {
	if s != nil {
		log.Printf("Error in NATS connection: %s: subscription: %s: %s", nc.ConnectedUrl(), s.Subject, err)
		return
	}

	log.Printf("Error in NATS connection: %s: %s", nc.ConnectedUrl(), err)
}

// called after reconnection
func reconnectHandler(nc *nats.Conn) {
	log.Printf("Reconnected to %s", nc.ConnectedUrl())
}

// called after disconnection
func disconnectHandler(nc *nats.Conn, err error) {
	log.Printf("Disconnected from NATS: %s", nc.LastError())
}
```

Calling `NewConnection()` will set up a new NATS connection that will forever reconnect and gets configured from the environment.

|Environment|Description|Required|Example|
|-----------|-----------|--------|-------|
|NATS_URL   |Servers to connect to|yes|`nats://n1.my.new:4222,nats://n2.my.net:4222`|
|NATS_CREDS |NATS 2.0 credentials to authenticate with||`/etc/fshipper/nats.creds`|
|NATS_CERTIFICATE|Public certificate to use for TLS||`/etc/fshipper/cert.pem`|
|NATS_KEY|Private key to use for TLS||`/etc/fshipper/key.pem`|
|NATS_CA|Certificate Authority chain for TLS||`/etc/fshipper/ca.pem`|

This helper is used for all connections in both the Producer and the Consumer, so it's worth making it robust and configurable.

We added 1 other little utility there related to `^C` handling.

## Producing log lines

Let's look at the producer; this is a stand-alone application compiled into a single binary and uses the above configuration variables and adds a few of its own:

|Environment|Description|Required|Example|
|-----------|-----------|--------|-------|
|SHIPPER_FILE|The file to publish|yes|`/var/log/system.log`|
|SHIPPER_SUBJECT|The NATS subject to publish to|yes|`shipper`|

```go
package main

import (
	"log"
	"os"
	"time"

	"github.com/hpcloud/tail"
	"github.com/nats-io/nats.go"

	"github.com/ripienaar/fshipper/internal/util"
)

func publishFile(source string, subject string, nc *nats.Conn) error {
	t, err := tail.TailFile(source, tail.Config{Follow: true})
	if err != nil {
		return err
	}

	log.Printf("Publishing lines from %s to %s", source, subject)

	for line := range t.Lines {
		nc.Publish(subject, []byte(line.Text))
	}

	return nil
}

func main() {
	source := os.Getenv("SHIPPER_FILE")
	if source == "" {
		log.Fatalf("Please set a file to publish using SHIPPER_FILE\n")
	}

	subject := os.Getenv("SHIPPER_SUBJECT")
	if subject == "" {
		log.Fatalf("Please set a NATS subject to publish to using SHIPPER_SUBJECT\n")
	}

	nc, err := util.NewConnection()
	if err != nil {
		log.Fatalf("Could not connect to NATS: %s\n", err)
	}

	for {
		err = publishFile(source, subject, nc)
		if err != nil {
			log.Printf("Could not publish file: %s", err)
		}

		time.Sleep(500 * time.Millisecond)
	}
}
```

That's a basic starting block for shipping the file, every time this starts it reads the file and sends it's entire contents and then follows it forever - even through log rotations. It's not perfect, but it's a start, we don't want to get lost in details of file tailing here (remember, use an off the shelve tool for real).

You can test this by running the binary using `SHIPPER_FILE=/var/log/system.log SHIPPER_SUBJECT="shipper" NATS_URL="localhost:4222" ./producer` and using `nats sub shipper` to verify it works.

## Consuming log lines

Let's create a Consumer. We'll listen on a subject and write everything we receive directly to a file; the file rotates daily. The consumer is a bit more complicated there's a lot of setup and `^C` handling and so forth.

This too is configurable using the environment:

|Environment|Description|Required|Example|
|-----------|-----------|--------|-------|
|SHIPPER_OUTPUT|The file to write to|yes|`/var/log/remote/system.log` or `/var/log/remote/system.log.%Y%m%d`|
|SHIPPER_SUBJECT|The NATS subject to publish to|yes|`shipper`|

Here the `SHIPPER_OUTPUT` takes a pattern and it will write log lines to that, it will rotate daily and keep a weeks worth (something to make configurable ideally).

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"strings"
	"time"

	rotatelogs "github.com/lestrrat-go/file-rotatelogs"
	"github.com/nats-io/nats.go"

	"github.com/ripienaar/fshipper/internal/util"
)

func main() {
	subject := os.Getenv("SHIPPER_SUBJECT")
	if subject == "" {
		log.Fatalf("Please set a NATS subject to consume using SHIPPER_SUBJECT\n")
	}

	output := os.Getenv("SHIPPER_OUTPUT")
	if output == "" {
		log.Fatalf("Please set a file to write using SHIPPER_OUTPUT\n")
	}

	ctx, cancel := context.WithCancel(context.Background())
	done := make(chan struct{})

	// start consuming messages until ctx interrupt
	err := consumer(ctx, done, subject, output)
	if err != nil {
		log.Fatalf("Consuming messages failed: %s", err)
	}

	for {
		select {
		case <-util.SigHandler():
			log.Println("Shutting down after interrupt signal")
			cancel()
		case <-done:
			log.Println("Shut down completed")
			return
		}
	}
}

func setupLog(output string) (*rotatelogs.RotateLogs, error) {
	// people can set their own file format, if no formatting characters are in the string we default
	if !strings.Contains(output, "%") {
		output = output + "-%Y%m%d%H%M"
	}

	return rotatelogs.New(output, rotatelogs.WithMaxAge(7*24*time.Hour), rotatelogs.WithRotationTime(24*time.Hour))
}

func consumer(ctx context.Context, done chan struct{}, subject string, output string) error {
	nc, err := util.NewConnection()
	if err != nil {
		log.Fatalf("Could not connect to NATS: %s\n", err)
	}

	log.Printf("Waiting for messages on subject %s @ %s", subject, nc.ConnectedUrl())

	out, err := setupLog(output)
	if err != nil {
		return err
	}
	defer out.Close()

	lines := make(chan *nats.Msg, 8*1024)

	_, err = nc.ChanSubscribe(subject, lines)
	if err != nil {
		return err
	}

	// save lined in the background forever till context signals
	go func() {
		for {
			select {
			case m := <-lines:
				fmt.Fprintln(out, string(m.Data))
			case <-ctx.Done():
				nc.Close()
				close(lines)
				done <- struct{}{}
				return
			}
		}
	}()

	return nil
}
```

Running this like `SHIPPER_SUBJECT=shipper SHIPPER_OUTPUT=/tmp/logfile NATS_URL=localhost:4222 ./consumer` will create `/tmp/logfile-202003180000` that rotates daily.

## Conclusion

So this is a very basic file Tail tool and a Consumer, it has several shortcomings:

 * 1 Producer/Consumer pair per host is required to split the lines by host
 * If you tried to scale the Consumer up, you start getting log lines out of order, and the file might get corrupt

So basically this is far from fit for purpose, but it does show we can publish data and receive it in a reasonably robust manner, and we can get our data to a central point.

Next posts we'll improve this to be better suited to our stated problem.
