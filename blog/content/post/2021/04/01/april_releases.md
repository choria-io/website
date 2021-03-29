---
title: "April 2021 Releases"
date: 2021-04-01T00:00:00+01:00
tags: ["releases"]
draft: false
---

We're pleased to announce the next set of Choria releases, these are mainly bug fixes, but we have a few important changes
to the Choria Server and Broker.

We have a new Registration plugin that will send all the data needed for discovery, previous supported plugin only read
a specific file regularly, the new plugin will send all the active state - facts, classes, collectives and more. This is
a first step towards building our own discovery database to replace our use of PuppetDB in the long run.

To configure the `inventory_content` registration plugin you can set:

```yaml
choria::server_config:
  registration: inventory_content
  plugin.choria.registration.inventory_content.target: mcollective.ingest.discovery.%{facts.fqdn}
  plugin.choria.registration.inventory_content.compression: true
```

Replacing `mcollective.ingest.discovery.%{facts.fqdn}` with your subject of choice. The intention is to ingest this into
our Streaming server - more detail below.

The Choria Broker is starting to use the NATS Account system to create isolation between different organisational units,
today we move all clients and nodes into a `choria` account as a first step.  If you are upgrading a cluster of Choria
Brokers expect to see some errors related to this account being unknown.  Once your entire cluster is upgraded it will
resolve. There might be some short network splits during this time.

Additionally, we now enable a new `system` account that will have events published in it for:

 * connects and disconnects
 * authentication errors
 * server shutdowns
 * regular server states

There are also a number of broker system level APIs for building reports and more. See the full post for details.

We're starting to expose a [NATS JetStream](https://docs.nats.io/jetstream/) based Streaming system, which we'll call
Choria Streaming, to ingest registration, scout status, system events and more for downstream processing and analysis.

This is a huge topic, one that we're still working on for Choria framing so more details on that later, this release
adds a number of configuration items related to that already.

The Puppet modules are now able to configure something called Leaf Nodes to facilitate access to Choria from remote offices
and, especially, high latency destinations.  A blog post will be published this week covering that.

Special thanks to Romain Tartière, Trey Dockendorf, Tim Meusel and Mark Frost for their contributions in this release.

<!--more-->
To access the system services configure a user like this, you'll still need to access the broker with the usual certs
and so forth, this is an identity to gain access just to the Choria Broker internals:

```yaml
choria::broker::system_user: system
choria::broker::system_password: secret
```

Once enabled the [natscli](https://github.com/nats-io/natscli) can be used to access the state of your network, here 
are some examples.

A login event:

```nohighlight
% nats event --context system.ams
Listening for Client Connection events on $SYS.ACCOUNT.*.CONNECT
Listening for Client Disconnection events on $SYS.ACCOUNT.*.DISCONNECT
Listening for Authentication Errors events on $SYS.SERVER.*.CLIENT.AUTH.ERR

[10:00:16] [brYbLCYu7vqIrXr5AxFWZ5] Client Connection

   Server: broker-broker-1.broker-broker-ss.choria.svc.cluster.local
  Cluster: CHORIA

   Client:
                 ID: 1619
               Name: dev1.devco.net
            Account: choria
    Library Version: 0.6.2  Language: ruby2
               Host: 10.244.0.23
```

A report of current cluster state:

```nohighlight
% nats server list --context system.ams
+------------------------------------------------------------------------------------------------------------------------------------------------------+
|                                                                   Server Overview                                                                    |
+-----------------+------------+----------------------+-----------+----+-------+------+--------+-----+---------+-----+------+------------+-------------+
| Name            | Cluster    | IP                   | Version   | JS | Conns | Subs | Routes | GWs | Mem     | CPU | Slow | Uptime     | RTT         |
+-----------------+------------+----------------------+-----------+----+-------+------+--------+-----+---------+-----+------+------------+-------------+
| broker-broker-1 | CHORIA     | choria.example.net   | 2.2.1.RC7 | no | 27    | 634  | 2      | 0   | 1.6 GiB | 1.0 | 0    | 3d14h9m33s | 65.451337ms |
| broker-broker-2 | CHORIA     | choria.example.net   | 2.2.1.RC7 | no | 48    | 690  | 2      | 0   | 288 MiB | 3.0 | 0    | 3d14h9m45s | 65.396505ms |
| broker-broker-0 | CHORIA     | choria.example.net   | 2.2.1.RC7 | no | 18    | 669  | 2      | 0   | 232 MiB | 5.0 | 0    | 3d14h8m59s | 65.308651ms |
+-----------------+------------+----------------------+-----------+----+-------+------+--------+-----+---------+-----+------+------------+-------------+
|                 | 1 Clusters | 3 Servers            |           | 0  | 93    | 1993 |        |     | 2.1 GiB |     | 0    |            |             |
+-----------------+------------+----------------------+-----------+----+-------+------+--------+-----+---------+-----+------+------------+-------------+

+----------------------------------------------------------------------------+
|                              Cluster Overview                              |
+---------+------------+-------------------+-------------------+-------------+
| Cluster | Node Count | Outgoing Gateways | Incoming Gateways | Connections |
+---------+------------+-------------------+-------------------+-------------+
| CHORIA  | 3          | 0                 | 0                 | 93          |
+---------+------------+-------------------+-------------------+-------------+
|         | 3          | 0                 | 0                 | 93          |
+---------+------------+-------------------+-------------------+-------------+
```

Detailed information about one specific server:

```nohighlight
% nats server info broker-broker-1 --context system.ams
Server information for broker-broker-0.broker-broker-ss.choria.svc.cluster.local (NB67UUCFQJRXKWTQM7UQX4VA5CWF2GKQV6NDD2JUYVWSMHKCADVV2D7Q)

Process Details:

         Version: 2.2.1.RC7
      Git Commit:
      Go Version: go1.16.2
      Start Time: 2021-03-25 19:52:10.394338822 +0000 UTC
          Uptime: 3d14h11m12s

Connection Details:

   Auth Required: false
    TLS Required: true
            Host: choria.example.net:4222
     Client URLs:

JetStream:

   Storage Directory: /data/jetstream
          Max Memory: 2.9 GiB
            Max File: 6.3 GiB
      Active Acconts: 1
       Memory In Use: 0 B
         File In Use: 1.1 GiB
        API Requests: 195
          API Errors: 48

Limits:

        Max Conn: 50000
        Max Subs: 0
     Max Payload: 1.0 MiB
     TLS Timeout: 2s
  Write Deadline: 10s

Statistics:

       CPU Cores: 2 4.00%
          Memory: 1020 MiB
     Connections: 19
   Subscriptions: 1
            Msgs: 6,720,721 in 7,939,818 out
           Bytes: 1.9 GiB in 2.5 GiB out
  Slow Consumers: 0

Cluster:

            Name: CHORIA
            Host: :::5222
            URLs: broker-broker-1.broker-broker-ss.choria.svc.cluster.local:5222
                  broker-broker-2.broker-broker-ss.choria.svc.cluster.local:5222
```

And much more like reports on all established connections and more, see the help from the `nats server --help` command, and it's README about configuration contexts.

## [Choria Server version 0.21.0](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.20.2...v0.21.0), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.21.0)

### Enhancements

 * Add a new registration plugin that sends the running inventory rather than file contents
 * Support enabling listening `pprof` port 
 * Restore the data plugin report in `rpcutil#inventory`
 * Create a choria account in NATS, move all connections there, enable system account
 * Add a `machine_state` data plugin
 * Support retrieving a single choria autonomous agent state using `choria_util#machine_state`
 * Support building ppc64le EL7 and EL8 RPMs
 * Drop support for Enterprise Linux 6 due to go1.16

### Bug Fixes

 * Fix validation for integers in the DDLs
 * Fail choria facts when no nodes match supplied filters
 * Do not send the filter verbatim in `choria req`
 * Add a client specific `TLSConfig()`, improve adapters and federation support for legacy certs
 * Correctly calculate advertise URL
 * Improve support for Clustered JetStream
 * Improve ping response calculations in federated networks
 * Avoid unnecessary warning level logs
 * Correctly detect stdin discovery
 * Improve stability of `choria scout watch`

## [choria/choria version 0.23.0](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.22.2...0.23.0), [Release](https://forge.puppet.com/choria/mcollective_choria/0.23.0/readme)

 * Support configuring leafnode connections
 * Support latest puppetlabs/stdlib
 * Enable access to the NATS system account
 * Support configuring Streaming
 * Add `server_service_enable` parameter
 
## [choria-mcorpc-support gem version 2.24.2](https://rubygems.org/gems/choria-mcorpc-support)

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.24.1...2.24.2), [Release](https://rubygems.org/gems/choria-mcorpc-support/versions/2.24.2)

### Enhancements

 * Drop dependency on win32-dir gem

### Bug Fixes

 * Relocate task cache locations to improve multi script support

## [choria/mcollective version 0.13.2](https://forge.puppet.com/choria/mcollective)

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.13.1...0.13.2), [Release](https://forge.puppet.com/choria/mcollective/0.13.2/readme)

### Enhancements

* Support latest `puppetlabs/stdlib`

### Bug Fixes

 * Fix mcollective fact on windows
 * Fix logfile path for windows

## [choria/mcollective_agent_bolt_tasks version 0.20.1](https://forge.puppet.com/choria/mcollective_agent_bolt_tasks)

Links: [Changes](https://github.com/choria-plugins/mcollective_agent_bolt_tasks/compare/0.20.0...0.20.1), [Release](https://forge.puppet.com/choria/mcollective_agent_bolt_tasks/0.20.1/readme)

 * Do not include .git files in the archive

## [choria/mcollective_agent_puppet version 2.4.1](https://forge.puppet.com/choria/mcollective_agent_puppet)

Links: [Changes](https://github.com/choria-plugins/mcollective_agent_puppet/compare/2.4.0...2.4.1), [Release](https://forge.puppet.com/choria/mcollective_agent_puppet/2.4.1/readme)

 * Support Puppet 7

## [choria/mcollective_choria version 0.20.2](https://forge.puppet.com/choria/mcollective_choria)

Links: [Changes](https://github.com/choria-plugins/mcollective_choria/compare/0.20.1...0.20.2), [Release](https://forge.puppet.com/choria/mcollective_choria/0.20.2/readme)

 * Do not include .git files in the archive

