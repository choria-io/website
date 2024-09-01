---
title: "September 2024 Releases"
date: 2024-09-01T00:00:00+01:00
tags: ["releases"]
draft: false
---

After a summer break we're back with a lot of new releases.

For the Choria Server we've made a number of improvements to our Autonomous Agents subsystem like enhancing the metric watcher to be able
to fire FSM events based on thresholds on metric data.

We also moved a step closer to a Puppet-free future with support for Choria Server deploying plugins into itself based on KV data at runtime.
While this was previously possible it is now built into the binary and can be enabled by the Provisioner.

The Provisioning system had updates that allow multiple active-active Provisioners to be run concurrently when managing version 0.29.4 of Choria.

For Puppet users there is a big change in our Puppet modules. Huge thanks to Romain Tartière who spent a lot of his holiday
time working on this, below his words about the update.

We updated the structure of the choria plugins.

While some plugins had a regular puppet module layout, other had an history with roots before this layout even existed, and where using some custom tooling for packaging.
This tooling allowed packaging these plugins as puppet modules, but also as .deb and .rpm packages as usually used in most Linux distributions.
Installing from these native system packages is something from a past era, and it does not make sense to continue supporting this.
Also, the custom layout we used prevented users from using the latest version of a plugin by pointing r10k to the git repository of the module.

For these reasons, we reworked the layout of the plugins so that they all use the standard structure of puppet modules.
It should not change anything for most users as modules on the Puppet forge are the same as before, and we have not been shipping .deb and .rpm modules anymore for a long time.
If you ever wanted to use the latest code of a module and got frustrated by the pain it was to do so, just be aware that now you can do this the way you do it with other modules.

We hope this change will help users and contributors to iterate more quickly on new features development and bug fixes.

In addition to the modules mention in the section below the following Puppet modules have been updated using the work Romain did:
`choria/mcollective_util_actionpolicy`,`choria/mcollective_agent_filemgr`, `choria/mcollective_agent_iptables`,`choria/mcollective_agent_nettest`,
`choria/mcollective_agent_nrpe`,`choria/mcollective_agent_package`, `choria/mcollective_agent_process`,`choria/mcollective_agent_puppet`,
`choria/mcollective_agent_puppetca`,`choria/mcollective_agent_service` and `choria/mcollective_agent_shell`.

Thanks to Trey Dockendorf, Dennis Ploeger, Romain Tartière and Vincent Janelle for their support in making these releases possible.

<!--more-->
## [Choria Server version 0.29.4](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.28.0...v0.29.4), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.29.4)

## Enhancements

* Support being called as `abt`
* Pass federation name to external discovery agent
* Adds a new `expression` watcher that can react to values stored in autonomous agent data
* Allow an individual `metric` watcher to disable Prometheus integration
* Support storing metric values in autonomous agent data
* Support publishing metrics to Graphite from the `metric` watcher
* Allow the `scout watch` command to ignore some autonomous agents
* Create a built-in agent and autonomous agent plugin service to support non CM deployments
* Send `alive` events every 30 minutes instead of every 1 hour
* Redesign the gossip service discovery for upcoming NATS 2.11 due June 2024
* Adds `skip_trigger_on_reenter` to the `scheduler` watcher to avoid some duplicate triggers
* Support for Debian Bookworm
* Adds `choria tool sha256` to compute recursive checksums compatible with `archive` and `plugins`
* Miscellaneous fixes and UX improvements for the `archive` watcher
* Support a `disown` setting in exec that ensures executed commands run after Choria stops
* All concurrent provisioners by maintaining a provisioner-lock on the agent

## Bug Fixes

* Use correct private inboxes for `scout watch` to support protocol v2 deployments
* Ensure the duplicate window aligns with the kv TTL when creating buckets

## [choria-mcorpc-support gem version 2.26.4](https://rubygems.org/gems/choria-mcorpc-support)

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.26.3...2.26.4), [Release](https://rubygems.org/gems/choria-mcorpc-support/versions/2.26.4)

### Enhancements

* Use GitHub Actions
* Fix Slack integration to use POST
* Allow overriding federation as cli argument and playbook func args

## [choria/mcollective version 0.14.6](https://forge.puppet.com/choria/mcollective)

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.14.5...0.14.6), [Release](https://forge.puppet.com/choria/mcollective/0.14.6/readme)

### Bug Fixes

* Fix catalog when systemd is not available

## [choria/mcollective_choria version 0.22.2](https://forge.puppet.com/choria/mcollective_choria)

Links: [Changes](https://github.com/choria-plugins/mcollective_choria/compare/0.22.1...0.22.2), [Release](https://forge.puppet.com/choria/mcollective_choria/0.22.2/readme)

* Support new module layout and tooling

## [choria/choria version 0.31.0](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.30.3...0.31.0), [Release](https://forge.puppet.com/choria/puppet-choria/0.31.0/readme)

* Support v2 protocol security when creating KV and Governors using Puppet types
* Switched to the static key URL to optimize key downloads
* Allow `choria::task::run` to catch errors
* Support v2 provisioning connections
* Fix protocol v2 issuers and security
* Fix adapters data type