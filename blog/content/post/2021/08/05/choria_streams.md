---
title: "Introducing Choria Streams"
date: 2021-08-05T00:00:00+01:00
tags: ["documentation"]
draft: streams
---

Choria Broker is based on the excellent NATS Server technology, this technology has been instrumental to moving Choria
from its MCollective roots where 1 000 managed nodes required a big hardware investment to where we are today with a
$40 Linode being enough to manage 50 000 nodes in an easy to manage and run single binary package.

NATS Server recently introduced a new capability called NATS JetStream and today I want to show a bit where we are with
making that available to Choria users as Choria Streams.

JetStream is a Streaming Server that uses a WAL to create an append-only log of messages. Messages get stored to disk
or memory, can be replicated within a cluster and can later be consumed by different consumers using any of the 40+
programming languages supported by NATS.

By embedding this technology in the Choria Broker we enable a number of use cases around our Metadata processing features,
Autonomous Agents, CloudEvents as produced by Choria Scout, and we also introduce 2 major new features: Choria Key-Value
Store and Choria Concurrency Governor.

This will all be available in our upcoming `0.23.0` release.

Read the full entry for an overview of where we are.

<--more-->

## Streams and Consumers

Streams are data stores full of messages, each stream listens to some subjects within the Choria broker and consume all the
messages from those subjects into the Stream. There are many settings to consider ranging from how long, or how many, messages
to keep to how many replicas of the messages to keep.

Choria Streams enable some Streams out of the box - captures [Lifecycle Events](https://choria.io/blog/tags/lifecycle/),
Autonomous Agent Events, Scout Statuses and Stream Admin API advisories.

Streams, compared to just core NATS subjects, are unique in that they store historical values and multiple defined Consumers
can read data from the stream, each Consumer having its own unique view over the stored data.

For Choria when considering providing data to other systems our focus will largely focus around Choria Streams. Examples are
Monitoring Statuses from Scout, Fleet Metadata, Fleet status updates and more, essentially anywhere we wish to make data available 
outside the confines of Choria we will do so via the Streams component.

This is a huge topic, for full details review the [JetStream Documentation](https://docs.nats.io/jetstream).  All the features 
available in JetStream is also available in Choria Streams. Client libraries for [Go](https://github.com/nats-io/nats.go),
, [Java](https://github.com/nats-io/nats.go), [C](https://github.com/nats-io/nats.c), [JavaScript, TypeScript and NodeJS](https://github.com/nats-io/nats.js)
already exist and more will come soon. Despite no specific support for a particular language any of the 40+ languages that support
NATS can interact with Choria Streams.

## Key-Value Stores

Key-Value Stores are the core component that makes much of the modern Cloud Native infrastructure possible.  Choria Key-Value Store
is a KV built on the Choria Streams technology. This means the Buckets can be replicated, have history, support mirroring across
regions and much more.

We made it available today in the CLI and in Autonomous Agents, full details in the [Key-Value Store Documentation](https://choria.io/docs/streams/key-value/).

Creating a bucket:

```nohighlight
# create bucket CFG, storing 5 historic values per key, replicated across 3 broker.
$ choria kv add CFG --history 5 --replicas 3
CFG Key-Value Store

     Bucket Name: CFG
         History: 5
             TTL: 0s
 Max Bucket Size: -1
  Max Value Size: 10240
```

Accessing a bucket:

```nohighlight
$ choria kv put CFG auth.username bob
bob
$ choria kv put CFG auth.password tooManySecrets
tooManySecrets
$ choria kv get CFG auth.username
CFG > auth.username created @ 05 Aug 21 10:55 UTC

bob

$ echo "${AUTH_USERNAME:?}"
zsh: AUTH_USERNAME: parameter not set
$ AUTH_USERNAME=$(choria kv get CFG auth.username --raw)
$ echo "${AUTH_USERNAME:?}"
bob
```

Much more is supported including watches and Autonomous Agents can watch for changes and make those changes available
to `exec` watchers today.

There will also soon be client libraries for multiple languages so that other non-choria systems can gain access to this data.

## Choria Concurrency Governor

Network wide concurrency control is a very difficult problem, long time users might recall our old `mco puppet runall 5` tool that
would attempt to actively orchestrate an entire fleet of nodes ensuring only 5 Puppet runs would be happening at any given time.

This tool was very resource intensive and placed a great toll on the brokers, often too much in the days before Choria Broker.

Choria Concurrency Governor uses the capabilities of Choria Streams to solve this problem much more elegantly, with a much lighter
impact on the network and no central orchestration.

As an example of this, lets set up a cron job to run `puppet agent --onetime --no-daemonize --no-usecacheonfailure --no-splay`.
Traditionally this was quite hard, and the only hacky solutions was to spread the start time out by some random factor, not too 
elegant and does not actually control the concurrency.

Lets set up a governor:

```nohighlight
$ choria governor add PUPPET 50 20m 3
Configuration:

  Capacity: 50
   Expires: 20m0s
  Replicas: 3
```

We create a Governor called `PUPPET` that allows 50 concurrent Puppet runs, any run has an allowance of 20 minutes to complete
else it will be evicted from the Governor - thus avoiding crashing software from rendering the Governor inoperable.

Now we can set up a cron job that fires our Puppet, network wide, at the same time:

```nohighlight
*/30 * * * * /bin/choria governor run \
    --config /etc/choria/server.conf \
    'PUPPET' \
    --max-wait 20m \
    '/opt/puppetlabs/bin/puppet agent --onetime --no-daemonize --no-usecacheonfailure --no-splay'
```

There will now never be more than 50 concurrent runs. We don't need to worry that we start it all at the same time, the
`choria governor run` command will actively campaign to get an execution slot and only when it does will it start Puppet.
Here, it will give up trying to find a slot after 20 minutes.

The CLI can be used to look at the leases in the Governor - which would include FQDNs of active nodes - and it lets you do
things like evict one or all of the entries from the Governor.

Governors can be used in `exec` Watchers in Autonomous Agents making it really easy to create rolling updates that can happen
without active orchestration via the RPC system.

## Message Submission System

Choria has always been a system that for any 1 request expect a single reply from any Agent that processed the request.
Agents cannot send a stream of regular updates, logs or other such data in addition to the initial response.

This is quite annoying and something I am looking to address using Streams. Additionally, as users focus development around
data in Streams they might soon find other information would make sense to have in he streams also.  

For example imagine out `mco tasks` command that fires off a Puppet Task in the background in a separate process. Wouldn't it be nice
if that task could send us its logs or notify us when it completes? The problem is such notifications would require connections
to the Choria Broker network, additional certificates, configuration management and all sorts of pain.

With Streams and Choria Servers new Message Submission feature any software on a node can send data to the Chroia Streams system,
cron jobs, agents, other services, anything you want.

The Choria Server will take ownership of delivering the message to Streams - it will retry delivery, ensure ordering, reuse its
existing mTLS TCP connection to the Choria Broker network and more.  All this means a single shot cron job can reliably with retries
and reliability handling submit messages to a Stream.

Once enabled this is as easy as:

```nohighlight
$ choria tool submit \
    --config /etc/choria/server.conf \
    --reliable \
    --sender "${HOSTNAME} Task Agent" \
    "tasks.status.${HOSTNAME}" /tmp/sdslksf.tmp  
```

This sends the contents to `/tmp/sdslksf.tmp` first into a disk spool owned by Choria Server where it will then, on the behalf
of the caller, deliver that message to a Stream.  This is a `reliable` message meaning it will be retried until the Stream
accepts it (with default max retries and expiry TTL).

Full documentation can be found in [Message Submit Documentation](https://choria.io/docs/streams/submission/).

We have a package for Go that lets you interact with this programmatically.

## Conclusion

This has been a piece of work that started late 2019 when we first started to support JetStream in our [data adapter system](https://choria.io/blog/post/2019/12/06/jetstream_adapter/). 

Streams has been available in Choria for about a year now but I was keeping it below the radar while the technology evolves,
and the picture becomes clear in my mind for how Choria will use it.

We are not done and will add many additional features around Streams, but I felt we are now at a time when I can start 
mentioning the work we are doing to bring this exciting technology to Choria users.

Keep an eye on the [Choria Streams Documentation](https://choria.io/docs/streams/) and I look forward to hearing feedback
from users. We hope to release the next Choria release in the coming days.

