---
title: "August 2021 Releases"
date: 2021-08-24T00:00:00+01:00
tags: ["releases"]
draft: false
---

This is the first release since April, and it's a massive release bringing many enhancements and new features.

We are introducing [Choria Streams](https://choria.io/docs/streams/) - a Stream Processing framework built into the Choria Broker
powered by [NATS JetStream](https://docs.nats.io/jetstream). I wrote a blog post about this [Introducing Choria Streams](https://choria.io/blog/post/2021/08/05/choria_streams/) that's worth a read.

Additionally, we added [Choria Key-Value Store](https://choria.io/docs/streams/key-value/), [Choria Governor](https://choria.io/docs/streams/governor/) and
[Choria Message Submit](https://choria.io/docs/streams/governor/) all powered by Choria Streams and each in their own right a big feature.

Other major enhancements are that we now support Websockets for the network connections between Servers, Broker and Go clients.

Autonomous Agents now have a data layer meaning within an Autonomous Agent data can be fetched from stores like other Key-Value 
stores and this data can be accessed by Watchers at run time. We expose node facts to Autonomous Agents in the data layer. 
Additionally, we support watching Choria Key-Value Store for changes which updates the data layer and trigger transitions. 
Exec Watchers also support Governors to create orchestration-free rolling upgrades etc.

We made huge improvements to Provisioning, we blogged about this in [Provisioning HA and Security](https://choria.io/blog/post/2021/08/13/secure_and_ha_provisioning/).
There you can also see we support Leader Election against Choria Streams as a library feature.

On the documentation front we added a big section about [Choria Streams](https://choria.io/docs/streams/) but also received permission to Open Source
some documentation that shows how a very large - millions of nodes - Choria deployment might look. This is a proven design in active use in production 
for a few years already.  We are busy building another such network at the moment, and a lot of the enhancements in Provisioning is as a result of 
this work.  Find the document at [Large Scale Design](https://choria.io/docs/concepts/large_scale/).

Thanks to Chris Boulton, Romain Tartière, Tim Meusel, Dominic Vallejo, Vincent Janelle and Franciszek Klajn for their contributions to this release

<!--more-->
## [Choria Server version 0.23.0](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.22.0...v0.23.0), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.23.0)

### Enhancements

 * Improve DDL data types for core DDL files
 * Allow the Choria Server to run in an Services-Only mode
 * Support Websockets for connectivity from Leafnodes and Choria Server to Choria Broker, also Go clients
 * Initial implementation of the `choria_registry` service agent
 * Adds a `choria login` command that supports delegating to `choria-login` in `PATH`
 * Improve sorting of `choria inventory` columns
 * Fail when a client cannot determine its identity
 * Allow the default collective to be set at compile time
 * Allow the default client suffix to be set at compile time (eg. rip.mcollective user id)
 * Allow a random sleep at the start of schedules for the Schedule watcher
 * Rate limit fast transitions in autonomous agents
 * Use default client-like resolution to find brokers in the JetStream adapter when no urls are given
 * Introduce [Choria Submission](https://choria.io/docs/streams/submission/) to allow messages to be placed into Streams via Choria Server
 * Support PKCS8 containers
 * Introduce [Choria Governor](https://choria.io/docs/streams/governor/) for network wide concurrency control
 * Support Governors in the Exec Autonomous Agent watcher
 * Additional Prometheus statistics for [Choria Streams](https://choria.io/docs/streams/)
 * Add a Autonomous Agent level data store, allow Exec Watchers to gather and store data in a Auto Agent
 * Allow Exec Watchers to access node facts
 * Add a [Choria Key-Value Store](https://choria.io/docs/streams/key-value/) accessible using `choria kv` and a new `kv` Autonomous Agent Watcher
 * Expose `kv` data to the Autonomous Agent data system
 * Support templates in Exec Watcher `cmd`, `env` and `governor`
 * Export certificate expiry time in Choria status files, support checking from CLI and Scout
 * Support Asynchronous Request mode in generated Go clients
 * Extend the RPC Reply structure to include what action produced the data
 * Use correct Choria reply subjects when interacting with the Streams API
 * Improve the broker shutdown process to cleanly shut down Choria Streams
 * Allow compiled-in Go agents to access the Submission system
 * Rename the `jetstream` adapter to `choria_streams`
 * Disable RPC Auth during provisioning mode
 * Support entering provisioning mode when the supplied `server.conf` does not exist
 * Generated clients can accept a Choria Framework, avoiding config loading etc
 * Include the time a RPC Reply was generated in the reply
 * Include the Public Key in the CSR reply, add data type hints to the provisioner DDL and update client
 * Support receiving private keys from the provisioner, protected using Curve 25519 ECDH shared secrets
 * Correctly enter provisioning with a configuration file and without a Puppet installation
 * Ensure SSL Cache is created if needed during provisioning
 * Support sorting `choria req` output by identity using `--sort`
 * Enable the `choria_provision` agent when provisioning is supported
 * Support Debian 11

## Bug Fixes

 * Fix setting workers and expr filter on generated clients
 * Ensure no responses list and unexpected responses list always prints, capped to 200 nodes

## [choria-mcorpc-support gem version 2.25.1](https://rubygems.org/gems/choria-mcorpc-support)

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.24.3...2.25.1), [Release](https://rubygems.org/gems/choria-mcorpc-support/versions/2.25.1)

### Enhancements

 * Pass path to active configuration file to choria

### Bug Fixes

 * Fix running bolt tasks on Windows
 * Only add AIO bin directory to PATH if it exists
 * Fix spawning of execution_wrapper.exe

## [choria/mcollective_choria version 0.21.1](https://forge.puppet.com/choria/mcollective_choria)

Links: [Changes](https://github.com/choria-plugins/mcollective_choria/compare/0.20.2...0.20.3), [Release](https://forge.puppet.com/choria/mcollective_choria/0.20.3/readme)

 * Update versions of the Ruby support system

## [choria/choria version 0.23.1](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.23.1...0.25.0), [Release](https://forge.puppet.com/choria/mcollective_choria/0.25.0/readme)

 * Support websocket ports in Choria Broker
 * Use EL8 repo for Fedora
 * Support configuring core Stream replica configuration
 * Support puppetlabs/apt 8.x
 * Prepare autonomous agents for data storage
 * Rename `jetstream` adapter to `choria_streams`
 * Support allowing provisioning against a core Choria Broker

## [Choria Server Provisioner version 0.12.0](https://github.com/choria-io/provisioner)

Links: [Changes](https://github.com/choria-io/provisioner/compare/v0.10.0...v0.12.0), [Release](https://github.com/choria-io/provisioner/releases/tag/v0.12.0)

**NOTE:** This project was renamed from `provisioning-agent` to `provisioner` since the agent is included in Choria Server now.

## Enhancements

 * Large code base refactor around generated clients
 * Support running in HA mode using Choria Streams for leader election
 * Support provisioning against the main Choria Broker, remove embedded broker support
 * Support provisioning Private Keys protected using Curve 25519 ECDH key exchange
