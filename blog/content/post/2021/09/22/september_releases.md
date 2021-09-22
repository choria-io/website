---
title: "September 2021 Releases"
date: 2021-09-22T00:00:00+01:00
tags: ["releases"]
draft: false
---

Today we're releasing the next Choria Server and a few Puppet modules. Primarily this is a bug fix and general improvement release with few real big ticket user facing items.

We have a major breaking change relating to our Package Repositories. For most people who use our public repositories nothing will change, but those using internal mirrors should probably read the full post for details.  In short, we are moving from Package Cloud to our own infrastructure hosted in EU, UK and US. Our packages and repositories are now signed using our own keys.

We've had some great feedback on [Choria Governors](https://choria.io/docs/streams/governor/) and we've improved the CLI tooling a bit, we've also added a new Puppet Type and Provider to manage these. Thanks to users who have been testing these new features.

We have an opt-in new feature that should significantly improve the default broadcast based discovery system.  Usually we wait for 2 seconds for discovery results, but in most cases most discovery results came in within the first few 100ms. By setting `plugin.choria.discovery.broadcast.windowed_timeout=1` in your client configuration file we now do a windowed discovery that will terminate if after the last received result no more results were received in 300ms. In most cases this will be a massive improvement in UX. Please test it, we aim to flip this to default on in near future.

We've had a big set of refactors on the Debian packaging and should have functioning Debian Bullseye packages for this release.  There's also been a few improvements to the Debian packages in general.

We have started the process of supporting a new style of agent called a Choria Service. These services will be used to perform AAA signing over the NATS protocol, to facilitate DDL free clients thanks to central Schema Registries and more. Today this is mainly under the cover improvements but expect big changes coming soon in areas of client deployment simplification.

Thanks to Romain Tartière, Romuald Conty and Tim Meusel for their contributions to this release

<!--more-->
## [Choria Server version 0.24.1](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.23.0...v0.24.1), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.24.1)

### Enhancements

 * Adds a helper to assist in creation of Governors from automation tools              
 * Allow provisioning of Action Policies and Open Policy Agent Policies via Choria Provisioner
 * Support listing known Governors                
 * Add `--force` / `-f` to `choria governor add`
 * Add a `splay` option to the Timer Watcher
 * Various refactors of Debian packages to behave more consistently with RedHat startup/restart flows
 * Introduce a faster broadcast discovery timeout using sliding windows, behind a opt-in setting
 * Allow Autonomous Agents to be compiled into the server as plugins
 * Initial support for performing AAA Server signing requests via Choria Services rather than HTTPS
 * Internal refactoring to improve cross/cyclic package import problems

### Bug Fixes

 * Do not attempt to also load embedded Autonomous Agents from disk
 * Do not create unconfigured Governors when viewing a non existing Governor
 * Create the `plugin.choria.machine.store` directory if it does not exist
 * Do not update file mtime on skipped checks in the File watcher
 * Handle JSON data in data better in Autonomous Agent data layer allowing for nested lookups
 * Fix logging of embedded NATS Server to Choria logs

## [choria/choria version 0.26.2](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.25.0...0.26.2), [Release](https://forge.puppet.com/choria/mcollective_choria/0.26.2/readme)

The Choria project is moving to using its own infrastructure for package hosting. We have a set of 3 servers - UK, EU and US - configured via mirror lists to replace our previous Package Cloud infrastructure. Oldest Ubuntu and Debian does not support mirror lists properly so they default to using the EU server, all others will use mirror lists. 

As part of this change all packages and repositories are now signed using our own GPG keys - which this module will take care of for you.

In the past you could influence the behaviour a bit using `choria::repo_gpgcheck` and `choria::repo_baseurl`, these settings have been removed.  If you use a local mirror please set `choria::manage_package_repo` to `false` and create your own repository management resources.

Governors can now be [managed using a Puppet resource](https://choria.io/docs/streams/governor/):

```puppet
choria_governor { 'puppet':
  capacity => 3,
  expire => 1200,
  replicas => 3,
  collective => 'mcollective',
  ensure => 'present',
}
```

 * Arch Linux: Use /usr/bin/ruby-2.7 instead of default Ruby 3
 * Arch Linux: adjust package name from choria to choria-io
 * Support new package repositories
 * Adds a type and provider to manage Governors

## [choria-mcorpc-support gem version 2.25.2](https://rubygems.org/gems/choria-mcorpc-support)

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.25.1...2.25.2), [Release](https://rubygems.org/gems/choria-mcorpc-support/versions/2.25.2)

 * Handle failures in the Delegated discovery system

## [choria/mcollective version 0.13.4](https://forge.puppet.com/choria/mcollective)

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.13.3...0.13.4), [Release](https://forge.puppet.com/choria/mcollective/0.13.4/readme)

 * Update rubypath to Ruby 2.7 on Arch Linux
 * Arch Linux: Ensure plugin directory exists
