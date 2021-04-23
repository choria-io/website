---
title: "April 2021 Releases"
date: 2021-04-23T00:00:00+01:00
tags: ["releases"]
draft: false
---

We're pleased to announce the next set of Choria releases, these are bug and feature releases.

We're starting to add the concept of a Service to Choria, a Service is a special kind of Agent that rather than requiring
discovery and handling multiple results will only ever have 1 response. The Agents hosted as Services will form a load
balanced group with High Availability and Reliability being the focus. 

We will use these to create node inventory services, configuration services for Scout and eventually also move our
AAA signing over to this format so that no TCP ports are needed other than the brokers. Foundational level features are being
released today, but we are still working on the big picture here.

We have recreated the long broken `choria plugin doc` command and move the `choria tool generate` commands also into
`choria plugin`, for `plugin doc` invoking the `mco` equivalent will call into Choria, but the old generate commands in `mco`
was too different so invoking those will now fail inviting you to invoke `choria plugin generate` instead. Plugin documentation
has been reformatted to look a bit nicer and now also support generating Markdown format output.

We updated our underlying NATS Server to version 2.2.2 which brings many stability and feature improvements.
The main feature is a system called JetStream that is already enabled within Choria - though more on that at a later stage
as we refine our particular use cases. If you wish to explore JetStream within Choria please reach out to us on the
usual community channels.  

A huge feature for us is that Websocket support has landed in the NATS. Today we do not yet expose these ports in Choria
but I'd love to hear from the community who would prefer this rather than our traditional TCP ports. 

Read all about NATS 2.2.2 on its [announcement blog post](https://nats.io/blog/nats-whats-new-22/).

Special thanks to Romain Tartière for his contributions in this release.

<!--more-->
## [Choria Server version 0.22.0](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.21.0...v0.22.0), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.22.0)

### Enhancements

* JetStream Adapter can publish to wildcard streams with per identity subjects
* Default to the choria account for leafnodes
* Support the old boolean_summary aggregator and generic output name remapping in summary aggregator
* Enable new Go based action policy by default
* Support wider duration specification by supporting week, month, year etc
* Create choria plugin doc and move tool generate to plugin generate
* Import the provisioning agent into this code base since it's now always compiled in
* Autonomous Agent transitions now support a human friendly description
* Initial support for Service Agents

### Bug Fixes

* Use correct target for registration messages
* Fix ordering of leafnode and accounts setup
* Improve consistency of time durations in ping output
* Increase leafnode authentication timeout
* Improve startup logs when skipping agents in specific providers
* Handle filter expressions that are not obviously boolean better

## [choria-mcorpc-support gem version 2.24.3](https://rubygems.org/gems/choria-mcorpc-support)

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.24.2...2.24.3), [Release](https://rubygems.org/gems/choria-mcorpc-support/versions/2.24.3)

### Enhancements

* Remove Ruby based code generators
* Set the environment variable PT__task
* Improve UX for users of choria.use_srv_records

### Bug Fixes

* Align MCollective::Util::Choria options processing with choria
* Avoid exceeding the stack when logging setup fails

## [choria/choria version 0.23.1](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.23.0...0.23.1), [Release](https://forge.puppet.com/choria/mcollective_choria/0.23.1/readme)

* Update to the latest Choria Server

## [choria/mcollective_choria version 0.20.3](https://forge.puppet.com/choria/mcollective_choria)

Links: [Changes](https://github.com/choria-plugins/mcollective_choria/compare/0.20.2...0.20.3), [Release](https://forge.puppet.com/choria/mcollective_choria/0.20.3/readme)

* Update versions of the Ruby support system

