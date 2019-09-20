---
title: "September 2019 Releases"
date: 2019-09-20T9:09:00+01:00
tags: ["releases"]
draft: false
---

Today we are doing a series of long overdue Puppet Module releases as well as a reasonably significant Choria Server update.

Significantly this delivers the new [External Agents](https://choria.io/blog/post/2019/09/18/external_agents/) I blogged about recently.

Read on for all the details!

<!--more-->

## Thanks

Thanks to Alexander Hermes, Steven Pritchard, Paul Tittle, Vincent Janelle and Ben Roberts for making these releases possible.

## Choria Server 0.12.1

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.12.0...v0.12.1), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.12.1)

This is quite a big release with significant new features:

### Enhancements

 * [External Agents](https://choria.io/docs/development/mcorpc/externalagents/) allow you to write agents in any language
 * RPC Authorization built into the Go Server thats compatible with *action_policy* (off by default)
 * A new tool to generate DDL files `choria tool generate ddl x.json x.ddl`, it can also convert JSON to Ruby

To enable the new Choria Server based authorization just set `rpcauthorization: 1` in your hiera data, it is off by default since it applies to all agents including any existing Ruby ones.  I invite the community to test this in their labs, we'll set it to on by default in the next release.

## choria/mcollective_agent_shell version 1.0.4

Links: [Changes](https://github.com/choria-plugins/shell-agent/compare/1.0.3...1.0.4), [Release](https://forge.puppet.com/choria/mcollective_agent_shell/1.0.4/readme)

### Enhancements

 * Improve compatibility with the pure JSON Choria Server

## choria/mcollective version 0.10.0

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.9.2...0.10.0), [Release](https://github.com/choria-io/puppet-mcollective/releases/tag/0.10.0)

### Notes

If you used `mco plugin package --format aiomodulepackage --vendor acme` in the past after this update you need to use `mco plugin package --vendor acme` as the `aiomodulepackage` has been merged into the support gem and renamed.

### Enhancements

 * Use PDK to build modules
 * Write choria_util policies by default
 * Retire the AIO Module packager - merged into the choria-mcorpc-support gem

## choria/mcollective_choria version 0.16.0

Links: [Changes](https://github.com/choria-io/mcollective-choria/compare/0.15.0...0.16.0), [Release](https://github.com/choria-io/mcollective-choria/releases/tag/0.16.0)

### Enhancements

 * Allow playbook node sets to declare zero discovered hosts is ok with *empty_ok*
 * Switch to using JSON format by default

### Bug Fixes

 * Improve puppetdb SRV lookup port resolution

## mcorpc-ruby-support gem version 2.20.6

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.20.5...2.20.6), [Release](https://github.com/choria-io/mcorpc-ruby-support/releases/tag/2.20.6)

### Enhancements

 * Check *client_activated* property when loading DDLs
 * Add support for type in outputs
 * Move the `aiomodule` package into this module, retire others