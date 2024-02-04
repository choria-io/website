---
title: "February 2024 Releases"
date: 2024-02-03T00:00:00+01:00
tags: ["releases"]
draft: false
---

After a few months break on sabbatical and other changes we're back with some new releases.

The main focus here is Puppet 8 and Ruby 3.2 support and this has largely been a community effort which was very 
good to see.

While we're moving our Puppet support forward we are also working on a future independent of Puppet. You might 
be aware that we have a Provisioner that can guide unconfigured Choria instances to being fully managed instances. 
The Provisioner though is focussed on configuration only, not on deployment of agents and other plugins.

Over the last year or so we have started designing ways to deliver Autonomous Agents at runtime, we now also support
delivering External Agents in this manner. In this way by placing a manifest of agents in a Key-Value bucket Choria
Server can completely deploy itself and it's plugins. Next we'll either turn our Ruby agents into Extenral Agents or
add the same ability to Ruby agents.

This is a big milestone in being Puppet free as there are no single point that still requires Puppet to go from zero to
a full Choria environment. We have a bit to go here, but it's conceivable that we might switch our standard deploy 
to start using these mechanisms to deploy plugins and configure Choria. 

In the end we will still support a Puppet deployment method but it will become much simpler as we will only need to do
a small part of the overall heavy lifting using Puppet. We will also unlock full featured environments backed by 
Kubernetes and other hosting environments.

Thanks to Jeff McCune, Pieter Loubser, Trey Dockendorf, Ryan Dill, Mark Dechiaro and Romain Tartière, Vincent Janelle for 
their support in making these releases possible.

<!--more-->
## [Choria Server version 0.28.0](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.27.0...v0.28.0), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.28.0)

## Enhancements

* Allow clients to view the ACLs applied to their connections in various utilities
* Allow setting SRV domain using the `CHORIA_SRV_DOMAIN` environment variable
* Adds additional utilities to maintain autonomous agent plugin manifests under `choria machine plugins`
* Upgrade to NATS Server 2.10.x and updates the embedded `nats` command line
* Various improvements to audit logging and expose its settings in `choria tool config`
* Allow audit log ownership to be set using `plugin.rpcaudit.logfile.group` and `plugin.rpcaudit.logfile.mode`
* Allow those who embed Choria Server to get notified when it's ready using `RegisterReadyCallback()`
* Support verifying packed plugin specifications in `machine pugins` and `mms`
* Ensure stream users can access KV and Object stores
* Expose the client governor permission on the jwt cli
* Support using in-process connections for adapter communications
* Only validate ed25519 signed provisioner tokens using the Issuer flow, fall back for rsa signed tokens
* Adds a new `plugins` watcher that can manage auto agents and external rpc agents
* Support booleans, enums and more in the `rpc` builder command flags parsing
* Use a native sha256 checker rather than rely on OS provided binary in the `archive` watcher
* Support runtime reloading and relocation of external agents without restarting the server

## Bug Fixes

* Improve shutdown reliability by giving Stream brokers more shutdown grace
* Disable `appbuilder` on Windows
* Retry calls to streams that can fail in early election setup
* Timeout initial connection attempts while preparing embedded nats CLI connection
* Grant access to governor lifecycle events for clients with the governor permission
* Trim spaces in received kv data in order to determine if it's JSON data or not

## [choria/mcollective_agent_puppet version 2.4.3](https://forge.puppet.com/choria/mcollective_agent_puppet)

Links: [Changes](https://github.com/choria-plugins/puppet-agent/compare/2.4.3...2.4.3), [Release](https://forge.puppet.com/choria/mcollective_agent_puppet/2.4.3/readme)

### Enhancements

* Support Ruby 3.2

## [choria-mcorpc-support gem version 2.26.3](https://rubygems.org/gems/choria-mcorpc-support)

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.26.2...2.26.3), [Release](https://rubygems.org/gems/choria-mcorpc-support/versions/2.26.3)

### Enhancements

* Ruby 3.2 Support
* Allow setting SRV domain using `CHORIA_SRV_DOMAIN` environment variable

## [choria/mcollective version 0.14.5](https://forge.puppet.com/choria/mcollective)

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.14.4...0.14.5), [Release](https://forge.puppet.com/choria/mcollective/0.14.5/readme)

### Enhancements

* Disable rpcauthorization for shim layer
* Allow defining facts refresh as systemd timer
* Support Puppet 8
* Allow disabling facts refresh management

## [choria/mcollective_choria version 0.22.1](https://forge.puppet.com/choria/mcollective_choria)

Links: [Changes](https://github.com/choria-plugins/mcollective_choria/compare/0.22.0...0.22.1), [Release](https://forge.puppet.com/choria/mcollective_choria/0.22.1/readme)

* Add gem dependency for Puppet 8 support

## [choria/mcollective_agent_filemgr version 2.0.2](https://forge.puppet.com/choria/mcollective_agent_filemgr)

Links: [Changes](https://github.com/choria-plugins/filemgr-agent/compare/2.0.1...2.0.2), [Release](https://forge.puppet.com/choria/mcollective_agent_filemgr/2.0.2/readme)

### Enhancements

* Support Ruby 3.2

## [choria/mcollective_agent_package version 5.4.1](https://forge.puppet.com/choria/mcollective_agent_filemgr)

Links: [Changes](https://github.com/choria-plugins/package-agent/compare/5.3.0...5.4.1), [Release](https://forge.puppet.com/choria/mcollective_agent_package/5.4.1/readme)

### Enhancements

* Support Ruby 3.2

## [choria/choria version 0.30.3](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.30.1...0.30.3), [Release](https://forge.puppet.com/choria/puppet-choria/0.30.3/readme)

* Use modern APT keyrings on Debian family
* Ensure tasks from non-production environments actually work