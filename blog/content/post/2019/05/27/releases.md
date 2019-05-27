---
title: "May 2019 Releases"
date: 2019-05-27T9:00:00+01:00
tags: ["releases"]
draft: false
---

This months releases come a bit late as things have been moving slow while I worked on a major new feature called [Choria Autonomous Agents](https://choria.io/docs/autoagents/) which releases in MVP today.

Keep an eye out for a follow up blog post that details those.  Apart from that it's just general house keeping releases.

One thing is worth pointing out: **This is the last release of Choria modules that support Puppet prior to version 6**

<!--more-->
## Puppet 6 and newer only

As you know Puppet Inc released Puppet 6 a few months ago and dropped support for MCollective.  We have tried to support versions 5 and 6 but the resulting module complexity has proven a massive burden for us and our users.  Thus we will going forward only support Puppet 6 and newer.

You can pin your modules to older versions and use Puppet 5 if you must, but as of the next releases expect some big(ish) changes to simplify our modules and distribution methods as we settle into a single model for the Ruby support.

## Choria Autonomous Agents

Autonomous Agents or Choria Machine as we also call them are systems that run on your node continuously and unsupervised. They are designed to support various IoT functionality like managing HVAC system and all kind of other reactive node level management.

The [documentation covers usecases and samples in detail](https://choria.io/docs/autoagents/), we'll post a video introduction this week.

This release is a (very) MVP release, expect a lot of movement in this area in coming releases

## Choria Server 0.11.0

Links: [Changes](https://github.com/choria-io/go-choria/compare/0.10.1...0.11.0), [Release](https://github.com/choria-io/go-choria/releases/tag/0.11.0)

Apart from above mentioned Autonomous Agents this release adds a little utility `choria tool status` that you can use in your own monitoring scripts to interrogate the `choria-status.json` file.

It allows you to detect if your Choria Server is not connected to any broker, if it's not had messages within a certain period and if the status file is being updated. I wanted to avoid every one of us reinventing the JSON parsing logic especially for cases where shell scripts are desired, we'll add more checks in time.

### Bug Fixes

 * Improve error messages logged when invoking puppet to retrieve setting values fail
 * Force puppet environment to production to avoid failures about missing environment directories

### Enhancements

 * Retry SRV lookups on reconnect attempts
 * Support Choria Autonomous Agents
 * Add a basic utility to assist with creating deep monitoring `choria tool status`

## Choria Puppet Module 0.13.1

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.13.0...0.13.1), [Release](https://github.com/choria-io/puppet-choria/releases/tag/0.13.1), [Forge](https://forge.puppet.com/choria/choria/readme)

### Bug Fixes

 * Disable repo_gpgcheck for yum repos to enable mirrors to be made
 * Ensure APT repositories are managed before the packages

## Choria Golang MCollective RPC Subsystem 0.4.0

Links: [Changes](https://github.com/choria-io/mcorpc-agent-provider/compare/0.3.3...0.4.0), [Release](https://github.com/choria-io/mcorpc-agent-provider/releases/tag/0.4.0)

### Enhancements

 * Support Choria Autonomous Agents
 * Log discovery requests

## Golang Security Subsystem 0.4.0

### Enhancements

 * Check a user certificate before privileged certificates to hopefully spam the logs less
 * Only update user certificates if they change when `SecurityAlwaysOverwriteCache` is set

### Bug Fixes

 * Support `SecurityAlwaysOverwriteCache` in the Puppet provider

## Ruby Choria Plugins 0.15.0

### Enhancements

 * Support Choria Autonomous Agents

### Bug Fixes

 * Fix path expansion for token locations with `~` in them

## Choria Provisioner 0.4.4

### Bug Fixes

 * Check context during retries to speed up shutdown
 * Handle work queue overflow better

## Choria Stream Replicator 0.4.1

Links: [Changes](https://github.com/choria-io/stream-replicator/compare/0.3.1...0.4.1), [Release](https://github.com/choria-io/stream-replicator/releases/tag/0.4.1)

### Enhancements

 * Allow TLS to be enabled for only one side of the replication bridge