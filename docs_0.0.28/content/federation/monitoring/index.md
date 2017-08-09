+++
title = "Monitoring"
weight = 200
+++

Your *Federation Broker Instances* form a Cluster collectively called a *Federation Broker*.  Each *Federation Broker Cluster* has a name like *london*, *production*, *newyork* etc.

## Observing the running cluster

Each *Federation Broker Instance* publishes it's stats to your NATS middleware on *choria.federation.production.stats*, you can consume those yourself but Choria includes a utility to summarize those:

{{% notice tip %}}
It can take a while for all your instances to show up since they only publish their stats every 10 seconds
{{% /notice %}}

```bash
$ mco federation observe --cluster production
Federation Broker: production

Federation
  Totals:
    Received: 820  Sent: 9823

  Instances:
    1: Received: 420 (51.2%) Sent: 4895 (49.8%)
    2: Received: 400 (48.8%) Sent: 4928 (50.2%)

Collective
  Totals:
    Received: 9823  Sent: 22062

  Instances:
    1: Received: 4895 (49.8%) Sent: 11288 (51.2%)
    2: Received: 4928 (50.2%) Sent: 10774 (48.8%)

Instances:
  1: version 0.0.25 started 2017-03-25 09:42:32
  2: version 0.0.25 started 2017-03-25 09:42:34

Updated: 2017-03-27 14:29:49
```

Here you can see the offloading of work in action, the Federation received *820* messages but published to the collective *9823*.  In Direct Addressed mode the MCollective Client needs to publish a message for every node it wish to communicate with.  In a Federation it asks the Federation Broker Instances to do this duplication rather than itself, thus reducing client CPU usage and network utilization.  Should you run multiple instances like here the work of producing these messages are shared across the cluster in batches of 200.

## Interrogating a specific instance

When you start your instances you have the option to specify monitoring ports, if you do that it will listen on localhost for HTTP requests where it will provide the same stats published to the middleware:

{{% notice tip %}}
It can only listen on *localhost* for HTTP requests and is off by default
{{% /notice %}}

You can request the */stats* uri on the Federation Broker Instance and you'll get JSON like below:

```json
{
  "version": "0.0.25",
  "start_time": 1490618153,
  "cluster": "development",
  "instance": "1",
  "config_file": "/etc/puppetlabs/mcollective/federation/development_1.cfg",
  "status": "OK",
  "threads": {
    "stats_web_server": {
      "alive": true,
      "status": "sleep"
    },
    "stats_publisher": {
      "alive": true,
      "status": "sleep"
    },
    "collective_middleware_handler": {
      "alive": true,
      "status": "sleep"
    },
    "federation_middleware_handler": {
      "alive": true,
      "status": "sleep"
    },
    "federation_inbox_handler": {
      "alive": true,
      "status": "sleep"
    },
    "collective_inbox_handler": {
      "alive": true,
      "status": "sleep"
    }
  },
  "collective": {
    "source": "choria.federation.development.collective",
    "received": 193,
    "sent": 324,
    "last_message": 1490618423,
    "connected_server": "nats://nats1.ldn.example.net:4222",
    "work_queue": 0
  },
  "federation": {
    "source": "choria.federation.development.federation",
    "received": 12,
    "sent": 193,
    "last_message": 1490618423,
    "connected_server": "nats://nats2.fed.example.net:4222",
    "work_queue": 0
  },
  "/stats": {
    "requests": 4
  }
}
```
