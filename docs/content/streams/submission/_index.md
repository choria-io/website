+++
title = "Message Submit"
weight = 30
+++

When building systems related to Choria it might be desirable to submit data from those processes to Choria Streams without
having to maintain connections from each process to Choria Broker.  Additionally, dealing with interruptions to the connection
would mean every process have to do its own spool management, retries and more.

Choria Server allows messages to be submitted to it for delivery to Choria Streams, Choria Server will maintain priority
spools and manage retries, expiry and more on the behalf of other processes.

Now your cron jobs or one shot applications can submit data into a Stream for off-line processing and archival.

Messages submitted to Choria can be *reliable* or *unreliable*.  Unreliable messages are attempted once only, and we will
not wait for Acknowledgement from Choria Streams - so technically these do not require Streaming capability.

Reliable messages are retried a maximum number of times for a maximum period of time. Reliable messages are ordered on a 
per-priority basis.  With 50 messages in the priority 1 spool, if message 1 fails messages 2 to 50 will not be delivered or attempted
until message 1 was successfully delivered.

{{% notice tip %}}
This feature added in Choria v0.23.0
{{% /notice %}}

## Configuration

### Choria Server

To enable this feature set the following value in your Hiera configuration:

```yaml
choria::server_config:
  plugin.choria.submission.spool: /var/lib/choria/submission
  plugin.choria.submission.max_spool_size: 500 # the default
```

Once this is set and Choria Server restarted it will create the directory if needed and start listening for messages
there.

### Creating a Stream

As a security precaution submitting a message to the subject *myapp.audit* will in reality submit the message to *main_collective.submission.in.myapp.audit*. The *main_collective* is that configured in the server configuration file.

Lets create a stream that will receive all *myapp* related messages, we do this using the `nats` CLI, review steps in [Configuration](../configuration) on configuring that:

```nohighlight
$ nats stream add MYAPP
? Subjects to consume mcollective.submission.in.myapp.>
? Storage backend file
? Retention Policy Limits
? Discard Policy Old
? Stream Messages Limit -1
? Per Subject Messages Limit -1
? Message size limit -1
? Maximum message age limit 30d
? Maximum individual message size -1
? Duplicate tracking time window 2m
? Replicas 3
Stream MYAPP was created

...
```

Here we create a stream called *MYAPP* that listens on *mcollective.submission.in.myapp.>* and it will retain data for 30 days.

## Submitting messages

The `choria tool submit` command can publish messages to Choria:

```nohighlight
# echo '{"cnt":1}' > /tmp/payload
# choria tool submit --config /etc/choria/server.conf --reliable --sender myapp.production myapp.metrics /tmp/payload 
```

Or from STDIN:

```nohighlight
# echo '{"cnt":100}'| choria tool submit --config /etc/choria/server.conf --reliable --sender mypp.production myapp.metrics -- -
ea8918a1-d3d5-4598-9a15-8d737902dccd
```

This submits a message to Choria, it's marked as reliable meaning the message will be retried until Choria Streams acknowledge
the message is stored to disk.  Retries are managed by the *--reliable*, *--max-ttl* and *--tries* flags.

If the spool is full message submission will fail.

## Viewing messages

At this point the message should be in the Stream, we can view all messages in a pager:

```nohighlight
$ nats stream view MYAPP
[1] Subject: mcollective.submission.in.myapp.metrics Received: 2021-07-11T10:22:53+02:00

  Nats-Msg-Id: ea8918a1-d3d5-4598-9a15-8d737902dccd
  Choria-Created: 1625991772906612419
  Choria-Identity: dev1.devco.net
  Choria-Priority: 4
  Choria-Reliable: 1
  Choria-Sender: mypp.production

{"cnt":1}
```

We can see the message is there, and some additional headers are set pertaining to the message. From here any NATS JetStream
capable client can consume and act on the messages.
