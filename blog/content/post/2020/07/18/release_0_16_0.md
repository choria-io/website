---
title: "Choria Server 0.16.0"
date: 2020-07-07T09:00:00+01:00
tags: ["releases"]
draft: false
---

We had a release quite recently but I wanted to release a number of Scout related features to early adopters, these
releases are mainly focussed on Scout but includes a few bug fixes and new builds for Ubuntu Focal (20.04 LTS).

The big item here is that we have integrated [Goss](https://github.com/aelsabbahy/goss) into the Scout framework and
it can now run validations regularly. See the [Scout Goss](../scout_goss/) blog post for details.

You'll also notice a new agent - `scout` - on your nodes, this gives API access to interact with Scout checks on Choria
servers.

Additionally, we are starting to work on our documentation for Scout, an [initial cut of this is also published today](https://choria.io/docs/scout/),
this shows our Puppet integration, Prometheus integration and a bit about the events.

Thanks to Romain Tartière for contributions to these releases.

Read on for the full details.

<!--more-->
## [Choria Server version 0.16.0](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.15.0...v0.16.0), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.16.0)

 * Initial work on a Scout framework towards building a monitoring related distribution
 * Add a new scout agent and Golang client
 * Release packages for Ubuntu Focal (20.04 LTS)
 * Add helpers to parse complex data in generated clients
 * Include a snapshot of recent check states in published check events
 * Extract the generic result display logic from `choria req` into a reusable package
 * Support performing goss validation in the nagios autonomous agent
 * Restore the ability for DDLs to declare display formats for aggregate outputs
 * Add a `choria scout watch` command
 * Fix targeting a specific sub collective in the req command
 * Generated clients perform 2 discoveries per request
 * Improve using the supplied logger in generated clients
 * Avoid zombies when Ruby agents exceed their allowed run time

## [choria/mcollective version 0.10.5](https://forge.puppet.com/choria/mcollective)

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.10.4...0.10.5), [Release](https://forge.puppet.com/choria/mcollective/0.10.5/readme)

 * Support setting policies for the `scout` agent

## [choria/mcollective_choria version 0.18.0](https://forge.puppet.com/choria/mcollective_choria)

Links: [Changes](https://github.com/choria-io/mcollective-choria/compare/0.17.3...0.18.0), [Release](https://forge.puppet.com/choria/mcollective_choria/0.18.0/readme)

 * Support the `scout` agent in the choria discovery method
 * Copy `scout` DDL files

## [choria/choria version 0.18.0](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.17.0...0.18.0), [Release](https://forge.puppet.com/choria/choria/0.18.0/readme)

 * Add `choria::scout_checks` helper
 * Support creating a gossfile from hiera
