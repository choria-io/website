---
title: "December 2019 Releases"
date: 2019-12-07T09:00:00+01:00
tags: ["releases"]
draft: false
---

It's been a while since we had release announcements and it's been a surprisingly busy period.

The main focus here has been on a number of stability and bug fixes, we've had some users dig in really deep into various aspects of the system and a number of bugs were squashed.

Past the quality of life stuff I have started reworking Choria Server Provisioning which will set us on a path to having a good Puppet free story, I have some POCs lying around of a Kubernetes based Broker, CA, and Provisioner that will give a really smooth path forward - provisioning is now compiled in to the FOSS stack by default and can be enabled using a JWT token, more on that in a future post. 

We also include a Tech Preview of [NATS JetStream support](https://choria.io/blog/post/2019/12/06/jetstream_adapter/) and significantly moved our event formats over to [Cloud Events v1.0](https://choria.io/blog/post/2019/12/05/cloudevents_transition/) format.

Thanks especially go to Alexander Hermes for his deep dive into all aspects of the client side playbooks. Deep dives into a product and filing some tickets, discussing the model on slack etc it hugely time consuming and very often this kind of community contribution flies under the radar but I find it more valuable than code, huge props to Alexander.

Other shout outs to Ben Robert, Yury Bushmelev, Romain Tartière and Vincent Janelle
<!--more-->

## Choria Server

We're releasing Choria Server 0.13.0 with new packages for Enterprise Linux 8.

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.12.1...0.13.0), [Release](https://github.com/choria-io/puppet-mcollective/releases/tag/0.13.0)

### Enhancements

 * Add a tech preview JetStream adapter
 * Switch to CloudEvents v1.0 format for lifecycle events and machine events
 * Build RHEL 8 packages nightly and on release
 * Support Synadia NGS as a NATS server for Choria
 * Add `choria tool jwt` to create provisioning tokens
 * Allow `choria req` output to be saved to a file
 * Force convert a DDL from JSON on the CLI without prompts
 * Update NATS Server to version 1.2.1 (via go-network-broker v1.3.2)
 * Expose the path to the agent configuration to external agents (via mcorpc-agent-provider v0.9.0)
 * Support regular expressions in callerid matches in action policy (via mcorpc-agent-provider v0.9.0)
 * Enable Choria Provisioning Agent by default and expose the provisioning JWT (via provisioning-agent v0.6.0)

### Bug Fixes

 * Improve startup when embedding the server in other programs
 * Improve stability on a NATS network with Gateways
 * Improve the calculations of total request time in the `choria req` command
 * Improve handling for actions without any inputs in DDL validation (via mcorpc-agent-provider v0.9.0)
 * Bug fixes to Ruby DDL generation (via mcorpc-agent-provider v0.9.0)

## [choria/mcollective_agent_package version 5.2.0](https://forge.puppet.com/choria/mcollective_agent_package)

Links: [Changes](https://github.com/choria-plugins/package-agent/compare/5.1.0...5.2.0), [Release](https://forge.puppet.com/choria/mcollective_agent_package/5.2.0/readme)

### Bug Fixes

 * Fix return value for apt_update

### Enhancements

 * Add the ability to search for available packages

## [choria/choria version 0.15.0](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.14.0...0.15.0), [Release](https://forge.puppet.com/choria/choria/0.15.0/readme)

### Enhancements

 * Allow splitting services log into server and broker logs

### Bug Fixes

 * Improve resource ordering on debian systems
 * Remove unneeded files from the packaged module

## [choria/mcollective version 0.10.2](https://forge.puppet.com/choria/mcollective)

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.10.1...0.10.2), [Release](https://forge.puppet.com/choria/mcollective/0.10.2/readme)

### Enhancements

 * Update stdlib dependency

## [choria/mcollective_choria version 0.17.1](https://forge.puppet.com/choria/mcollective_choria)

Links: [Changes](https://github.com/choria-io/mcollective-choria/compare/0.16.1...0.17.1), [Release](https://forge.puppet.com/choria/mcollective_choria/0.17.1/readme)

### Bug Fixes

 * Align playbook log level names with Puppet

### Enhancements

 * Detect pure string results from Playbooks and render them correctly
 * Support both Puppet 5 and 6 paths for the task helper
 * Support using Synadia NATS NGS as a broker for Choria
 * Update the `choria-mcorpc-support` dependency

## [choria/mcollective_util_actionpolicy version 3.2.0](https://forge.puppet.com/choria/mcollective_util_actionpolicy)

Links: [Changes](https://github.com/choria-plugins/action-policy/compare/3.1.0...3.2.0), [Release](https://forge.puppet.com/choria/mcollective_util_actionpolicy/3.2.0/readme)

### Enhancements

 * Support regular expressions in callerid matches

## [choria-mcorpc-support gem version 2.20.8](https://rubygems.org/gems/choria-mcorpc-support)

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.20.7...2.20.8), [Release](https://rubygems.org/gems/choria-mcorpc-support/versions/2.20.8)

## Enhancements

 * Improve handling `--version` in applications
 * Update `nats` dependency to support multi tenancy