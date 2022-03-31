+++
title = "Monitoring"
weight = 150
toc = true
+++

This sections covers how you might monitor your Choria Server, Choria Broker and data held in Choria Streams.

## Choria Server

By default, Choria writes `/var/log/choria-status.json` when configured using Puppet, you can adjust this though:

| Setting                                | Description                                                        |
|----------------------------------------|--------------------------------------------------------------------|
| `plugin.choria.status_file_path`       | When set activates writing status data in JSON format to this file |
| `plugin.choria.status_update_interval` | The interval, in seconds, that the file will be written            |

We provide a NAGIOS format health check that can interrogate this file:

```nohighlight
$ choria tool status --help
usage: choria tool status --status-file=STATUS-FILE [<flags>]

Checks the health of a running Choria instance based on its status file

Flags:
  --help                     Show context-sensitive help (also try --help-long and --help-man).
  --version                  Show application version.
  --debug                    Enable debug logging
  --config=FILE              Config file to use
  --status-file=STATUS-FILE  The status file to check
  --disconnected             Checks if the server is connected to a broker
  --message-since=1h         Maximum time to allow no messages to pass (0 disables)
  --max-age=30m              Maximum age for the status file (0 disables)
  --certificate-age=24h      Check if the certificate expires sooner than this duration (0 disabled
  --token-age=24h            Check if the token expires sooner than this duration (0 disabled)
  --unprovisioned            Checks that the server is in provisioning mode
  --provisioned              Checks that the server is not being provisioned
```

Using this with your favorite monitoring tool you can monitor the health of the Choria Server,
its certificates, JWT tokens and more.

## Choria Broker

Choria Broker is a single process that manages the communications between Clients and Servers.
It's long-running and can be clustered.

### HTTP Based Monitoring

To enable HTTP based monitoring - required for curl, health checks, Prometheus metrics and more -
configure these settings:

| Setting                            | Description                                           |
|------------------------------------|-------------------------------------------------------|
| `plugin.choria.stats_port`         | The HTTP port to listen on for HTTP requests          |
| `plugin.choria.stats_address`      | A host address to listen on, defaults to 127.0.0.1    |
| `plugin.choria.network.pprof_port` | Enables go pprof debugging and profiling on this port |

Enabling the `stats_port` will enable the full [NATS Monitoring Capability](https://docs.nats.io/running-a-nats-service/nats_admin/monitoring),
accessible using curl or other HTTP client.

### Prometheus

Choria embeds a Prometheus Exporter into its Broker and add a number of Choria specific statistics,
with the `plugin.choria.stats_port` enabled this is accessible on `/choria/prometheus`. 

We have some published dashboards on the Grafana website.

### Cluster Based Monitoring

The Choria CLI has a number of monitoring tools that operate over the NATS protocol. There's some 
setup needed.

On your Choria Broker you have to configure a System User and Password:

```ini
plugin.choria.network.system.user = system
plugin.choria.network.system.password = s3cret
```

The Client configuration where you will monitor from will also need these settings.

#### Basic Broker and Cluster information

A number of tools will list and view details about the Brokers, here are some examples:

```nohighlight
$ choria broker server list
╭────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│                                                                  Server Overview                                                                   │
├─────────────────┬────────────┬──────────────────────┬────────────┬─────┬───────┬───────┬────────┬─────┬─────────┬─────┬──────┬──────────────┬──────┤
│ Name            │ Cluster    │ IP                   │ Version    │ JS  │ Conns │ Subs  │ Routes │ GWs │ Mem     │ CPU │ Slow │ Uptime       │ RTT  │
├─────────────────┼────────────┼──────────────────────┼────────────┼─────┼───────┼───────┼────────┼─────┼─────────┼─────┼──────┼──────────────┼──────┤
│ broker-broker-0 │ CHORIA     │ choria.example.net   │ 2.7.5-beta │ yes │ 22    │ 920   │ 2      │ 0   │ 234 MiB │ 2.0 │ 0    │ 11d20h56m46s │ 57ms │
│ broker-broker-2 │ CHORIA     │ choria.example.net   │ 2.7.5-beta │ yes │ 30    │ 901   │ 2      │ 0   │ 238 MiB │ 1.0 │ 0    │ 11d20h58m25s │ 57ms │
│ broker-broker-1 │ CHORIA     │ choria.example.net   │ 2.7.5-beta │ yes │ 48    │ 848   │ 2      │ 0   │ 258 MiB │ 2.0 │ 0    │ 20d2h34m52s  │ 57ms │
├─────────────────┼────────────┼──────────────────────┼────────────┼─────┼───────┼───────┼────────┼─────┼─────────┼─────┼──────┼──────────────┼──────┤
│                 │ 1 Clusters │ 3 Servers            │            │ 3   │ 100   │ 2,669 │        │     │ 729 MiB │     │ 0    │              │      │
╰─────────────────┴────────────┴──────────────────────┴────────────┴─────┴───────┴───────┴────────┴─────┴─────────┴─────┴──────┴──────────────┴──────╯

╭────────────────────────────────────────────────────────────────────────────╮
│                              Cluster Overview                              │
├─────────┬────────────┬───────────────────┬───────────────────┬─────────────┤
│ Cluster │ Node Count │ Outgoing Gateways │ Incoming Gateways │ Connections │
├─────────┼────────────┼───────────────────┼───────────────────┼─────────────┤
│ CHORIA  │ 3          │ 0                 │ 0                 │ 100         │
├─────────┼────────────┼───────────────────┼───────────────────┼─────────────┤
│         │ 3          │ 0                 │ 0                 │ 100         │
╰─────────┴────────────┴───────────────────┴───────────────────┴─────────────╯
```

Other commands to try:

 * `choria broker server info broker-broker-1` shows general information about a running server
 * `choria broker server report connections` lists and summarize connections
 * `choria broker server request connz` equivalent to requesting `/varz` over HTTP, but for the whole cluster or named server, see other `request` sub commands also
 * `choria broker serer report jetstream` overview report of JetStream / Choria Streams usage

#### Broker Health Checks

We have a number of NAGIOS protocol health checks to verify your cluster is functional.

##### Connectivity

Basic connectivity can be checked, this checks any of the configured middleware servers you might
want to create a configuration file listing each of your brokers and passing that with `---choria-config`:

```nohighlight
$ choria broker server check connection
OK Connection OK:connected to nats://choria.ams.devco.net:4222 in 2.438923ms OK:rtt time 61.979889ms OK:round trip took 0.061221s | connect_time=0.0024s;0.5000;1.0000 rtt=0.0620s;0.5000;1.0000 request_time=0.0612s;0.5000;1.0000
```

##### Broker Health

Server CPU, Memory, Connections, Subscriptions etc can be monitored, the `--name` argument is required,
so check all your Brokers individually. Connection is to any server in your middleware list.

```nohighlight
$ choria broker server check server --name broker-broker-1
OK broker-broker-1 | uptime=1737852.4866s cpu=1% mem=273547264 connections=48 subscriptions=848
```

##### Choria Streams Cluster

When running Choria Streams there is a cluster-wide RAFT group that manages the overall placement
and health of the system. We monitor this here for size and readyness.

```nohighlight
$ choria broker server check meta --expect 3 --seen-critical 1m --lag-critical 1000
OK JetStream Meta Cluster OK:3 peers led by broker-broker-1.broker-broker-ss.choria.svc.cluster.local | peers=3;3;3 peer_offline=0 peer_not_current=0 peer_inactive=0 peer_lagged=0
```

##### Choria Streams

Specific Streams can be monitored for various dimensions like numbers of messages, cluster size,
last message age and more.  Here's an example but see --help for details.

```nohighlight
$ choria broker server check stream --stream CHORIA_REGISTRATION --peer-expect 3 --msgs-warn 100
OK CHORIA_REGISTRATION OK:3 current replicas | peers=3;3;3 peer_offline=0 peer_not_current=0 peer_inactive=0 peer_lagged=0 messages=8350;100
```

##### Choria Key-Value Buckets

We can check that a bucket exists and has a specific value:

```nohighlight
$ choria broker server check kv --bucket CHORIA_AJ_ELECTIONS --key task_scheduler
OK CHORIA_AJ_ELECTIONS OK:bucket CHORIA_AJ_ELECTIONS OK:key task_scheduler found | values=1 bytes=185B replicas=3
```
