---
title: "January 2021 Releases"
date: 2021-01-13T09:00:00+01:00
tags: ["releases"]
draft: false
---

We have a number of releases today that will be the start of big changes in our modules. These releases will hopefully have a minor impact on users, but the next release or two will require some Hiera changes, so it's worth keeping an eye on these. **For the next while testing in your labs and dev environments is essential**.

This is the beginning of a big push to once again simplify our deployment story.  Choria started as a trivial way to install MCollective but things have changed quite a lot since then and unfortunately entropy has had its effect on our modules.

In addition to these changes we also have some pretty amazing additions to the Choria Servers.

Read on for the background and details of what's to come.

Special thanks to Tim Meusel, Vincent Janelle, Vladislav Kuspits and Romain Tartière for their contributions in this release.

<!--more-->
## WARNING

I'll start with a warning, this is a big change. Many files will be deleted from the lib dirs, many new files will be made. We're changing a default that will remove many unmanaged files.

You have to test this release in pre-release or staging, we did our best, we found many upgrade issues in development but we cannot anciticpate every possible scenario.  Please test thoroughly.

## Background

It's worth a quick look back at how we got here. Initially Choria was a way to install MCollective easily for Puppet 4 users. To do that we wrote some Ruby plugins for the Puppet Inc MCollective system as delivered in `puppet-agent`.

We had a `mcollective` module to handle the configuration aspects of MCollective and a `mcollective_choria` one to install our plugins into it. 

This was easy to use and easy to understand what's what. We had wide user adoption, lots of systems managed by these plugins and lots of code, hiera data and more relying on them.

Since then Puppet Inc deprecated their shipped MCollective, donated the old code back to me, and I wrote a new Go based server to replace `mcollectived`. To support this new server, its broker and various other components a `choria` module was created.

Today there's an unfortunate mix of mcollective and choria modules - with sometimes even contradicting and overlapping configuration options. For some things to work both modules need to be configured with the same settings.

We also have configuration and libraries in the old Puppetlabs locations, it would be better to move into our own locations.

This is not acceptable, in the end attempting to keep users systems working without big changes in their deployments gave us a confusing and difficult to use system.

## The way forward

We will rip the bandage off and get rid of the 2 mcollective related modules. This will have some unfortunate consequences for users wrt having to change their deployment and Hiera data but ultimately will be for the best for everyone. This is the first time we'll do a breaking change to users Puppet code really since the inception of this project, we'll be careful and hope to do it right, so we can give you another few years of stability after.

Todays releases starts this process by making a number of big changes. These will not yet require major changes to your processes - except maybe two items - but will set us up to achieve the goal of getting rid of the overlapping modules with competing configuration approaches.

 * MCollective plugins, DDLs etc are stored in `/opts/puppetlabs/mcollective/plugins/mcollective`. **As of this release files not managed by Puppet in this directory will be purged (after being filebucketed)**. This will help us prepare to move them to their own directory in the next release. If you place files there out of Puppet I suggest you [package them into modules](https://choria.io/docs/development/mcorpc/packaging/).
 * Configuration is moved from `/etc/puppetlabs/mcollective` to `/etc/choria`. We will try to purge from the old location, but you might have some left over there that can be removed
 * 50+ files from the `mcollective_choria` module was moved into the Gem, these will be purged from the libdir. `choria-mcorpc-support` version `2.23.1` will the lowest version we support.
 * With an eye to retiring the `mcollective_agent_bolt_tasks` module we moved the `mcollective_agent_bolt_tasks::ping` task to `choria::ping`

We have a number of `mco <command>` commands that are considered Core to MCollective - `inventory`, `choria_util`, `rpc` and more - we have reimplemented a number of these in the Go binary and in an effort to minimise the number of overlapping implementations some changes are being made here:

 * `mco rpc` will now automatically invoke `choria req` (aka `choria rpc`)
 * `mco inventory` will now automatically invoke `choria inventory`
 * We created a `mco find` equivalent in choria called `choria discover` (aka `choria find`) but not redirected this as it's an important debugging tool pf the core behaviors.
 * `mco choria_util request_cert` will error and instruct you to use `choria enroll` instead
 * The Go system got a PuppetDB discovery system and will honor the configuration of default discovery method, the `--dm` flag was added a few places to allow you to choose as before. 
 
Today these redirections are silent, in a new release they will also log warnings.

This means in order to use these `mco` commands you will now need the `choria` binary on your system. We made this easier by supporting setting `choria::server: false` in hiera and hope to release a Homebrew tap for OS X users to make the `choria` binary available there.

## Upcoming changes

In our next module releases the `choria` module will receive opt-in abilities to manage client configuration and the client management in the `mcollective` will support opt-out. This will let you migrate your configuration over at a time that's convenient.  After that we will remove the ability for the `mcollective` module to package these, which will then make it redundant.

The remaining features delivered by the `mcollective_choria` module will be accessed by the Gem, but we need some improvements in the Go daemon side.

We will produce a followup post that guides you in migrating from old Hiera values to the newer approach at the next release.

## Release Details

## [Choria Server version 0.19.0](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.18.0...v0.19.0), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.19.0)

### Overview

There is a number of really significant improvements to the Choria Server in this release, most significantly we support `-S` compound filters again but using a new language that's easy to extend and maintain.

We've adopted the [expr](https://github.com/antonmedv/expr) based Expression Language to build a new filtering system, this is in use in 2 places:

 * When performing requests using the `rpc` application the results can be filtered on the client side using an expression. See [Filtering results ](/docs/concepts/cli/#filtering-results)
 * When discovering nodes the compound filters can be used to perform complex matching against remote nodes. In a future release we'll support data plugins. See [Complex Compound or Select Queries](/docs/concepts/cli/#complex-compound-or-select-queries).

Both of these features are experimental and will only be 100% solid once Data plugins are supported. For now I'd be very keen on any feedback you might have.

### Enhancements

 * Use new JetStream enhancements to improve `choria scout watch` history retrieval
 * Add a `metrics` Autonomous Agent that can poll and publish metrics
 * Perform DNS lookups on each initial connection retry to improve handling early bootup scenarios
 * Major code cleanups and test coverage improvements in the Autonomous Agent Watchers
 * Allow Autonomous Agent Watchers to be plugins, convert all core ones to plugins, expose them in `choria buildinfo`
 * Ignore case when doing fact matching
 * Ignore case when matching against configuration management classes
 * Add a `choria_status` Nagios builtin allowing Choria to health checks from Scout
 * Avoid listening and registering with mDNS when Homekit is not used
 * Add `choria inventory`
 * Create Go clients for `rpcutil`, `scout` and `choria_util` in `go-choria/client`
 * Add a PuppetDB discovery method
 * Add `--dm` to the `choria req` command to switch discovery method
 * rpc client will now honor the `DefaultDiscoveryMethod` setting for all clients
 * Generated clients has a PuppetDB name source
 * Add `choria discover` aka `choria find`
 * Report the certificate fingerprint when doing `choria enroll` for Puppet CA
 * `choria ping` now calculates times from publish to reply-received and reports connection setup and security overhead separately
 * Support the `color` option in more places and disable it on Windows
 * Add `expr` based client-side filtering of RPC results, `rpc` command and generated clients
 * Basic support for Data plugin DDLs
 * Support `expr` based compound filters with `-S`
 * Improve consistency of discovery related cli options

## Bug Fixes

 * Improve support for HTTPS servers discovered by SRV records by stripping trailing `.` in names
 * Prevent nil pointer access in `choria enroll` when the private key already exist but no CSR

## [choria/mcollective_agent_puppet version 2.4.0](https://forge.puppet.com/choria/mcollective_agent_puppet)

Links: [Changes](https://github.com/choria-plugins/puppet-agent/compare/2.3.3...2.4.0), [Release](https://forge.puppet.com/choria/mcollective_agent_puppet/2.4.0/readme)

### Enhancements

 * Support Puppet 7
 * Add `--skip_tags` option
 * Have `-E` be a short version of `--environment`

## [choria-mcorpc-support gem version 2.23.1](https://rubygems.org/gems/choria-mcorpc-support)

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.22.1...2.23.2), [Release](https://rubygems.org/gems/choria-mcorpc-support/versions/2.23.1)

### Enhancements

 * Relax config parsing rules when parsing server configs to avoid failing on Choria Server configuration
 * Do not log deprecation warnings in configuration
 * Update rake, rspec, mocha, rubocop, modernise code style and test style
 * Support redirecting an application to an external command
 * Redirect `mco rpc` and `mco inventory`
 * Import `mcollective-choria` ruby code base
 * Allow tasks to run as another user
 * Remove `mco choria request_cert` in favor of `choria enroll`
 * Remove the old PuppetCA enroll code
 * Retire old compound filter support code and support new `expr` based compound filters
 * Parse user configuration locations in the same way as the go client

## Bug Fixes

* Pick the correct path prefix on Windows in line with Choria Server

## [choria/choria version 0.21.0](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.20.0...0.21.0), [Release](https://forge.puppet.com/choria/choria/0.21.0/readme)

### Enhancements 

 * Add a new defined type `choria::scout_metric` with a Hiera based collection defined type `choria::scout_metrics`
 * Remove references to defunct`mcollectived`
 * The `choria::scout_check` defined type can now accept additional properties that custom internal checks might support
 * Remove various EOL Ubuntu versions
 * Support YUM repositories for EL8
 * Puppet 7 support
 * Support disabling the server in a way that's compatible with PuppetDB discovery
 * Add `choria::ping` task, imported from `mcollective_agent_bolt_tasks`
 * Add dependencies that once was on `mcollective_choria` to this module
 * Manage additional directories that's required for the move of configuration to `/etc/choria`
 * Relocation configuration to `/etc/choria`
 * Retire support for Compound filters in the ruby shim
 * Fix NATS Streaming Server data adpater type

## [choria/mcollective version 0.13.0](https://forge.puppet.com/choria/mcollective)

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.13.0...0.13.0), [Release](https://forge.puppet.com/choria/mcollective/0.13.0/readme)

### Enhancements

 * Add an option to disable service management
 * Revert server facts removal - broke PuppetDB discovery
 * Relocate configuration into `/etc/choria`
 * Remove `mcollective::package`

## [choria/mcollective_choria version 0.17.3](https://forge.puppet.com/choria/mcollective_choria)

This repository [https://github.com/choria-io/mcollective-choria](https://github.com/choria-io/mcollective-choria) is being archived after its contents were merged into the Ruby gem and 2 small standalone modules were created.

### Enhancements

 * Cache private keys to avoid prompting many times in batch operations for passphrases
 * Support tasks with multiple files
 * Fix PuppetDB discovery for `choria::service`

## [choria/mcollective_choria version 0.19.0](https://github.com/choria-plugins/mcollective_choria/)

This is a new module with a few support files that used to be in `mcollective-choria`, it replaces the previous `choria/mcollective_choria` one. Being in a separate repository r10k and similar tools can now install the entire Choria module suite at arbitrary hashes.

### Enhancements

 * Perform package installs for all OSes of the MCORPC Ruby Support libraries
 * Support Puppet 7

## [choria/mcollective_agent_bolt_tasks 0.19.0](https://github.com/choria-plugins/tasks-agent)

This is a new module with a few support files that used to be in `mcollective-choria`, it replaces the previous `choria/mcollective_agent_bolt_tasks` one. Being in a separate repository r10k and similar tools can now install the entire Choria module suite at arbitrary hashes.
