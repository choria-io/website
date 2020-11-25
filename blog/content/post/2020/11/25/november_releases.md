---
title: "November 2020 Releases"
date: 2020-11-25T09:00:00+01:00
tags: ["releases"]
draft: false
---

We have a number of small releases today, mainly quality of life changes - performance improvements and such.

The only major work here is around our Autonomous Agent feature, this lets you build managed finite state machines
that can manage components on your machines without RPC interaction. This underpins our Scout checks and helps in
IoT scenarios etc.

Today we're adding 2 new watchers, an Apple HomeKit Button and a Timer. The HomeKit button is interesting in home
automation scenarios where a Choria Autonomous Agent can appear to your Apple devices as a button that you can toggle
from your Apple Home apps. Combined with the timer it's possible to create an override button for HVAC, Fans etc that interrupts
a normal managed schedule for a while. For example when watching a movie I don't like having my extractor fan on, using 
any Apple device I can now set a 2 hour override, after 2 hours normal scheduled activity resumes so I don't need to
remember to re-enable the extractor.

In future releases we'll add a Timer based maintenance window to Scout checks using the timer watcher.

We're starting to work on supporting Puppet 7, progress is being made (thanks Tim!) but I think we have some way to go.

Special thanks to Tim Meusel and Romuald Conty for their contributions in this release.

<!--more-->
## [Choria Server version 0.18.0](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.17.0...v0.18.0), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.18.0)

### Enhancements

 * Add an Apple HomeKit Button watcher
 * Add a Timer watcher

## [choria/choria version 0.20.0](https://forge.puppet.com/choria/mcollective)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.19.0...0.20.0), [Release](https://forge.puppet.com/choria/mcollective/0.20.0/readme)

### Enhancements

 * Add FreeBSD support for choria
 * Always deploy choria server
 * Support Ubuntu focal and KDE neon 20.04

## [choria/mcollective version 0.12.0](https://forge.puppet.com/choria/mcollective)

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.11.0...0.12.0), [Release](https://forge.puppet.com/choria/mcollective/0.12.0/readme)

### Enhancements

 * Improve module containment
 * Improve the Puppet datatypes
 * Support FreeBSD
 * Remove mcollective server facts
 * Improve network utilization by using file() and not source
 * Initial Puppet 7 support

## [choria-mcorpc-support gem version 2.22.1](https://rubygems.org/gems/choria-mcorpc-support)

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.22.0...2.22.1), [Release](https://rubygems.org/gems/choria-mcorpc-support/versions/2.22.1)

## Enhancements

 * Stop parsing registration related settings which would fail when Choria Server is configured
