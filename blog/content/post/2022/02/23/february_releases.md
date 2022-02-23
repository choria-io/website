---
title: "February 2022 Releases"
date: 2022-02-23T00:00:00+01:00
tags: ["releases"]
draft: false
---

It's been almost 5 months since our last release, not because nothing has been happening but because so much has been happening, good problems to have!

So this is a bit of a massive release, however I think the bulk of the changes will not affect our typical Puppet based users.

## Choria Registry

This introduces first work of a new Choria Registry. We have a long-standing pain point around managing DDL files on clients, it's a technical
requirement to describe remote services but it's just a pain to maintain, Puppet helps but for clients in CI, desktops etc, the DDL requirement
is just too much.

Choria Server now has an option to act as a Registry where it can read it's local DDL directory and serve that up to clients on demand.
When a client tries to access a new agent it has never accessed before it will ask the registry for the DDL describing that agent. It will
also do so regularly to ensure the local cache is still accurate.

This means that we can now have truly single-file client deployments. With just the `choria` binary and a running Registry that choria client
can interact with the entire fleet and do everything it wants. This is a great improvement for deployment of client machines and making Choria
more generally useful without Configuration Management.

The Choria Server can be a Registry, running multiple Servers with registry enabled will create a failure tolerant HA cluster of registry servers.

This is a brand-new feature, so I am not yet documenting it publicly, but I am keen to talk to users who wish to help in validating this before 
we look to supporting this more widely.

## Non mTLS communications

The major work here that contributed to the 20 000 line code change in Choria Server is that we now support a secure non mTLS mode
of communication. This is of no consequence for Puppet users so if that's you feel free to skip this section.

With a typical deployment we use the Puppet CA to create a fully managed and closed mTLS based network. For some enterprises replicating 
that with their internal PKI infrastructure is nearly impossible. So we looked to, optionally, move away from a pure mTLS mode to a mixed 
setup where we use ED25519 keypair and signed JWTs to provide equivalent security.

Essentially we now have formalized our use of JWT into a new `tokens` package where servers and clients have their own JWT. We hope to 
move entirely over to this model in time as we were able to create a greatly enhanced security model:

 * Servers are restricted to only certain collectives, attempting to enter non defined collectives will be denied by the broker
 * Servers are restricted to only server traffic flows. A server token cannot make a request to any other server, enforced by the broker
 * Servers have a default deny permission set allow specific access to Streams, Governors, Hosting Services and being able to be a Submission Server
 * Clients have private reply channels, clients cannot view each others replies
 * In addition to Open Policy Agent a set of default deny permissions allowing access to use Streams, administer Streams, use Elections, view Events, use Governors etc
 
Using these settings moves us to a much more secure and private setup where even between 2 Choria Users traffic is now isolated and secure and this
introduces the first of a security model around our adoption of Choria Streams. We cannot replicate these policies using just certificates. We hope 
to move even Puppet users to this model in future but that's a big undertaking to get right without additional services.

To enable these features one needs to deploy AAA Service and Provisioner - and both of those had recent releases supporting this mode.

As mentioned this is not really a thing that Puppet users should worry about however those in large enterprises who deploy in non-Puppet ways should 
keep an eye out for incoming documentation around this feature.

## Package Repository Changes

As notified back in September we are moving away from Packagecloud to our own package hosting infrastructure. I am keeping the Packagecloud
infrastructure up for a while but this release and all future ones will not be uploaded there to promote users from moving to the new infrastructure.

Thanks to Romain Tartière, Steffy Fort, Tim Meusel and Alexander Olofsson for their contributions to this release

<!--more-->
## [Choria Server version 0.25.0](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.24.1...v0.25.0), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.25.0)

## Removals

 * Remove NATS Streaming Server support

## Enhancements

 * Add a CLI API for managing KV buckets
 * Allow `choria scout watch` to show only state changes
 * Support asserting provisioning state in the health check plugin
 * Adds a new `archive` watcher to manage `tgz` files, not enabled by default
 * Adds a new `machines` watcher to manage Choria Autonomous Agents, not enabled by default
 * Refactor DDL resolution, support querying Choria Registry for unknown DDLs
 * Change docker base to AlmaLinux
 * Show additional `mco choria show_config` style information in `choria tool config`
 * Support `stdout` and `stderr` as logging destinations in addition to discard and a file name
 * Add SPDX License Identifier and Copyright to source files
 * Support tallying wildcard components rather than just a single component
 * Allow custom loggers to be passed to Choria and avoid changing settings of the default logrus logger
 * Support tallying governor events
 * Support for latest Cert Manager APIs
 * Add `--senders` to `choria req` that shows only those replying identities
 * Allow successful KV operations that do not change data to transition autonomous agents
 * Move to NATS official KV implementation, formalize Leader Election in Choria Broker
 * Allow non TLS connections from both servers and clients in combination of AAA and Provisioner using JWTs
 * Extract all jwt handling code in all packages into a new `tokens` package
 * Allow JWT clients to have permissions that can restrict access to Choria Streams related features
 * Extend provisioning agent to on board ed25519 seeds and process signed JWTs from the provisioner
 * Support enabling connection `nonce` feature allowing per connection private key validation
 * Import the nats CLI tool into Choria under `choria broker`
 * Specifically use `choria broker run` to start the broker
 * Unify the kv del and kv rm commands
 * Expand the `jwt` command to create other types of JWT and move to `choria jwt`
 * Allow custom builders to set the server service to auto start after install
 * Add 64 bit ARM packages
 * Support checking server JWT token validity

## Bug Fixes

 * Compatibility fix for 32 bit builds
 * Improve starting Choria Streams between reboots
 * Improve tool provision so debugging custom provisioning targets is more reliable
 * Correctly handle missing server configuration files when a custom provisioner is set
 * Ensure filters work with async requests in the choria req command 
 * Improve `choria tool governor run` when the broker is down
 * Relax identity validation in `flatfile` discovery to avoid rejecting some valid hostnames as identities
 * Ignore Autonomous Agents with `-temp` name suffix and the `tmp` directory
 * Compatibility fix for latest NATS Server code regarding dynamic limits

## [choria/choria version 0.27.1](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.26.2...0.27.1), [Release](https://forge.puppet.com/choria/mcollective_choria/0.27.1/readme)

NOTE this changes the monitoring interface to loopback by default, if you poll Prometheus stats
from your brokers set `choria::broker::stats_listen_address` to `::` as it now defaults to `::1`

 * Allow access to Choria::TaskResults#bolt_task_result
 * Add package_source parameter
 * Provide stats over the loopback address, allow the stats to be disabled entirely
 * Add choria::kv_buckets and choria::governors
 * Arch Linux: Switch from Ruby 2.7 to 3
 * Add the choria_kv_bucket resource
 
## [choria-mcorpc-support gem version 2.26.0](https://rubygems.org/gems/choria-mcorpc-support)

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.25.3...2.26.0), [Release](https://rubygems.org/gems/choria-mcorpc-support/versions/2.26.0)

 * Add a Choria::TaskResult#bolt_task_result API

## [choria/mcollective version 0.14.1](https://forge.puppet.com/choria/mcollective)

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.13.4...0.14.1), [Release](https://forge.puppet.com/choria/mcollective/0.14.1/readme)

 * Improve module metadata for operating systems and puppet versions
 * Arch Linux: Switch from Ruby 2.7 to 3
 * Add install_options for the gem install

## [choria/mcollective_choria version 0.21.3](https://forge.puppet.com/choria/mcollective_choria)

Links: [Changes](https://github.com/choria-plugins/mcollective_choria/compare/0.21.2...0.21.3), [Release](https://forge.puppet.com/choria/mcollective_choria/0.21.3/readme)

 * Update DDL files and dependencies
