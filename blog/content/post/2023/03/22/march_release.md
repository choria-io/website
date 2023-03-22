---
title: "March 2023 Releases"
date: 2023-03-22T00:00:00+01:00
tags: ["releases"]
draft: false
---

It's been a long time since our previous release in November as we have been working on some major architectural changes and new features.

This is quite a big release but the majority of changes will not affect Puppet users today.

## New Core Contributor

We have a new Core Contributor who some of you might recognise from MCollective days, please welcome [Pieter Loubser](https://github.com/ploubser). 

## New Docker Registry

Due to recent changes at the official Docker Hub we now have a new Docker Registry.  Please [read the announcement](https://choria.io/blog/post/2023/03/20/docker_changes/). No more containers will be pushed to the official Docker Hub.

While implementing this change we also activated automated nightly container builds for most components.

## New Project Websites

We have documentation for our overall distribution of Choria at [choria.io/docs](https://choria.io/docs) but this documentation is slanted heavily to the Puppet user - in essence choria.io documents a distribution of Choria for Puppet users. For more in-depth looks into what the components can do and features not exposed to Puppet users we launched a number of new websites:

 * [Choria Backplane](https://choria-io.github.io/go-choria/)
 * [Stream Replicator](https://choria-io.github.io/stream-replicator/)
 * [Provisioner](https://choria-io.github.io/provisioner/)
 * [AAA Service](https://choria-io.github.io/aasvc/)
 * [App Builder](https://choria-io.github.io/appbuilder/)

## App Builder Experiments

App Builder is our little no-code tool to build admin command line tools. We are experimenting with a project-level task mode that allow you to place a file in a directory and then run `abt`, which will then be a different command depending on where you are.

This is an experimental command, we'd love some feedback.  It's documented in the new [Experiments](https://choria-io.github.io/appbuilder/experiments/index.html) page.

## New Security Model and Protocol

We have introduced a new security model based on JWT files and ed25519 signatures that provide huge improvements for deployments at scale:

 * Based on ed25519 signed JWT files
 * A chain of trust between a new component called an Organization Issuer and various components allowing full delegation of credential issuance
 * Strong set of role based permissions for clients, servers, provisioners and unprovisioned nodes
 * Policy embedded in the JWT files for lighter touch server configuration - no more policy files on servers
 * Strict deny-all security on the Choria Brokers for much improved security and privacy of traffic
 * Vastly improved `choria jwt` command that includes monitoring of tokens and early integration with Hashicorp Vault
 * Redesigned the protocol moving on from some legacy decisions made in Marionette Collective

Longer term this will allow us to not rely on Certificate Authorities for identity and give us far greater control in how, when and for how long clients are enrolled.

In the process we had to do a lot of internal refactoring to the main Choria Framework and related systems like AAA Service, Provisioners, Replicators and more.

 * We wrote an [Architecture Decision Record](https://choria-io.github.io/go-choria/adr/001/index.html) describing the problems and goals of this work
 * We have an in-depth guide on testing and exploring the feature [V2 Protocol & Security](https://choria-io.github.io/go-choria/previews/protov2/index.html)
 * We have a [Docker Compose](https://github.com/ripienaar/choria-compose) environment that fully demonstrates the new model 

In time, as we complete some other efforts around delivery of RPC Agents into this system, we'll slowly move all users over to this model.  For now this should not concern Puppet users.

## Minor Server Features

[External Agents](https://choria.io/docs/development/mcorpc/externalagents/index.html) can now have multi-arch binaries allowing the same tar file to be deployed to a mix of servers when the agents require compilation.

We have a new output format in the `choria req` command enabled using the `--jsonl` flag, this will produce JSON Lines output for every major event.  Using this high quality wrapper libraries can be created in any language quite rapidly.  A ruby wrapper that supported progress bars, discovery and all other behaviors was less than 200 lines. We look forward to seeing what the Python users in our community do with this!

When a Choria Server is managed by the Choria Provisioner it now supports in-place over-the-air upgrades of itself at provisioning time.

We also landed many bug fixes and UX improvements to various `choria` commands.

Thanks to Romain Tartière and Pieter Loubser for their contributions to this release

<!--more-->
## [Choria Server version 0.27.0](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.26.2...v0.27.0), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.27.0)

## Enhancements

 * Introduce Choria JWT based security and Protocol version 2
 * Choria Message Submit can sign published messages when using Choria Security
 * Enhance the request signing protocol to include signatures made using the private key
 * Introduce the concept of a Organization Issuer and chain of trust JWT tokens for Server and Client issuers
 * Support Hashicorp Vault as storage for the Organization Issuer and the `choria jwt` command
 * Do not terminate servers on authentication error
 * New Client JWT permissions to indicate a client can access the `provisioning` account in the broker
 * Allow provisioning over non TLS when holding an Org Issuer signed provisioning JWT
 * Support Choria Provisioner using version 1 Protocol
 * Support full Choria version upgrades during provisioning
 * Add a new RPC Authorization plugin that requires and authorize policies found in client JWTs
 * Create a new dedicated backplane docs site https://choria-io.github.io/go-choria
 * Allow the `machines` watcher spec signer public key to be set in config
 * Support `direct mode` for Choria Key-Value Stores to increase scale and throughput
 * Support multi-arch binaries for external agents
 * Support streaming JSON output on `choria req` to assist non-golang clients to be built quicker
 * Create a tool to monitor JWT token health and contents
 * Add the `--governor` permission to `choria jwt server`
 * Include the number of Lifecycle events published in instance stats, data and rpcutil output
 * Record exec watcher events in lifecycle recorder
 * Emit new `upgraded` events when release upgrading a running server via provisioning
 * Support leader election for tally and label metrics by leader state
 * Support adding headers to Choria Message Submit messages
 * Record the builtin type as plugin in nagios watcher events

## Deprecations

 * Remove numerous deprecated configuration settings

## Bug Fixes

 * Improve handling defaults in output DDLs for generated clients
 * Improve fact filter parsing to handle functions both left and right of the equation
 * Ensure provisioning tokens have a default non-zero expiry
 * Improve DDL schema validation
 * Improve `plugin generate ddl` UX
 * Improve handling of governors on slow nodes and during critical failures
 * Fix validation of Autonomous Agents that use timer watchers
 * Allow `choria machine run` to be used without a valid Choria install
 * Correctly detect paths to ed25519 public keys that are 64 characters long as paths
 * Ensure multiple AAA Login URLs are parsed correctly

## Other Changes

 * Extract the tokens package into github.com/choria-io/tokens
 * Add `context.Context` to the provisioner target resolve `Configure()` method
 * Export `SetBuildBasedOnJWT` in default proftarget plugin

## [choria/choria version 0.29.0](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.28.4...0.29.0), [Release](https://forge.puppet.com/choria/choria/0.29.0/readme)

 * Update dependencies

## [choria/mcollective_choria version 0.21.5](https://forge.puppet.com/choria/mcollective_choria)

Links: [Changes](https://github.com/choria-plugins/mcollective_choria/compare/0.21.4...0.21.5), [Release](https://forge.puppet.com/choria/mcollective_choria/0.21.5/readme)

* Update DDL files and dependencies
