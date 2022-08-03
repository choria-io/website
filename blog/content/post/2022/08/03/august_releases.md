---
title: "August 2022 Releases"
date: 2022-08-03T00:00:00+01:00
tags: ["releases"]
draft: false
---

It's time for a minor release of Choria and a few related components, not a huge
release as we are preparing a big refactor but still some user facing improvements -
especially to the recently added App Builder component.

## New Election features

We've used leader elections for a while internally for various components, these are
backed by Choria Streams.

In this release we added an `choria election` command, using this command you can:

 * Manage, view, evict elections and leaders
 * Designate a particular node as a leader by managing a file under election using `choria election file`
 * Run a command under leader election using `choria election run`

See the [Election](https://choria.io/docs/streams/elections/) documentation.

## New Governor features

Governors can traditionally only control maximum concurrent executions, use this to
limit how many nodes are actively running Puppet for example.

With a small change we were able to make them support a mode of maximum executions per
period.

Imagine you have 10 nodes running the same cron job, if you need the job to run only
on one of the servers every hour - essentially creating a failover pool - you can use
the new `--max-per-period` flag.

See the [Governor](https://choria.io/docs/streams/governor/) documentation.

## Go Versions

The Go team released 1.19 last night, this means going forward we will support only
go 1.18 and 1.19. The `github.com/choria-io/go-choria` tag `v0.26.1` will be the
last to work on Go 1.17.

Special thanks to Tim Meusel for his contributions to this release!

<!--more-->
## [Choria Server version 0.26.1](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.26.0...v0.26.1), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.26.1)

## Enhancements

* Upgrade `appbuilder` to `0.3.0` with new `template`, `report` and `write_file` transforms
* Allow in-process connections to nats from the brokers, used to optimise Streams bootup
* Governors can control executions per period
* Adds `choria election` with various admin tools and tools to run commands and cron jobs under leader election
* Switch to a new more compact help template
* Support signing JWT tokens using ed25519 tokens
* Refactor protocol and security layers to start work on version 2 of the network protocol

## Bug Fixes

* Improved handling of ed25519 seed and jwt miss-matches during provisioning and startup
* Improved detection of STDIN being JSON data, avoiding unexpected switches to flat file discovery method under cron
* Improve reliability of managed autonomous agent cleanup
* Force gzip compression on Jammy debs to improve compatability with other distros and mirroring tools

## [choria/choria version 0.28.2](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.28.1...0.28.2), [Release](https://forge.puppet.com/choria/mcollective_choria/0.28.2/readme)

* Upgrade dependencies

## [choria/mcollective version 0.14.3](https://forge.puppet.com/choria/mcollective)

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.14.2...0.14.3), [Release](https://forge.puppet.com/choria/mcollective/0.14.3/readme)

* Improve facter support under PE
