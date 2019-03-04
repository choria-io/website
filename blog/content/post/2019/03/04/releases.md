---
title: "February 2019 Releases"
date: 2019-03-04T13:28:00+01:00
tags: ["releases"]
draft: false
---

I typically release around the 20th of the month, this one was a bit delayed while I worked with the NATS project on some problems we encountered. Nothing major in these releases as I have been traveling and working on a large implementation.

Some work that is not mentioned here is that I am reworking my Choria network load tester tool, this essentially allow you to use lets say 20 AWS instances to run a Choria network of 15 000 nodes.  It does this by starting multiple Choria Servers on a single node in Go routines and connecting them to the network in various formations.  This is ongoing, reach out to me if anyone has interest in this tool.  This focus is mainly to assist me in testing the upcoming NATS 2.0 release for uptake into the Choria Broker.

For Puppet users there is a potential big change to look out for, Choria has a stated goal of:

```nohighlight
Choria sets up the popular Action Policy based authorization and does so in a default deny mode which means by default, no-one can make any requests
```

There was a problem though in that any modules that had no explicit policies would end up being in default allow mode, this addressed across a few of these updates so you might need to keep an eye on this in your environment.

Special thanks to Romain Tartière and Konrad Scherer for their contributions during this cycle.

<!--more-->
## Choria Server 0.10.1

Links: [Changes](https://github.com/choria-io/go-choria/compare/0.10.0...0.10.1), [Release](https://github.com/choria-io/go-choria/releases/tag/0.10.1)

#### Bugfixes

On large networks - around 25 000 nodes or more - we observed some instability in the clustering of Choria Brokers, you would see log lines like these:

```nohighlight
rid:16993 - Slow Consumer Detected: WriteDeadline of 5s Exceeded
```

This indicates that the route connection (hence `rid` and not `cid`) between brokers would get disconnected. This turned out to be a regression upstream that I worked with the NATS team to resolve.

## Ruby Choria Plugins 0.14.1

Links: [Changes](https://github.com/choria-io/mcollective-choria/compare/0.13.1...0.14.1), [Release](https://github.com/choria-io/mcollective-choria/releases/tag/0.14.1), [Forge](https://forge.puppet.com/choria/mcollective_choria)

#### Enhancements

 * Add `-T` to the `federation trace` command
 * Improve error messages when a token file cannot be found
 * Disable TLS verify when speaking to signers as we are not guaranteed to have client certs
 * Ensure policy defaults are honored correctly

## MCollective RPC Ruby Support 2.20.5

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.20.4...2.20.5), [Release](https://github.com/choria-io/mcorpc-ruby-support/releases/tag/2.20.5), [Gem](https://rubygems.org/gems/choria-mcorpc-support)

#### Bug Fixes

 * Fix fact summaries for complex data types

## Choria Action Policy Authorization Plugin 3.1.0

Links: [Changes](https://github.com/choria-plugins/action-policy/compare/3.0.0...3.1.0), [Forge](https://forge.puppet.com/choria/mcollective_util_actionpolicy/readme)

#### Enhancements

 * Set `allow_unconfigured` to false by default

## Choria Puppet Module

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.12.0...0.13.0), [Release](https://github.com/choria-io/puppet-choria/releases/tag/0.13.0), [Forge](https://forge.puppet.com/choria/choria/readme)

#### Enhancements

 * Allow the Package Cloud repo to be mirrored and a local url used when configuring the repos