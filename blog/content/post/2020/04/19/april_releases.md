---
title: "April 2020 Releases"
date: 2020-04-19T09:00:00+01:00
tags: ["releases"]
draft: false
---

It's been quite some time since we've had releases and there's been a huge list of small improvements.

Thanks to those who contributed to these releases: David Gardner, Mark Frost, Romain Tartière, Yury Bushmelev, @rjd1, Tim Meusel, Alexander Hermes, Vincent Janelle

<!--more-->
## Choria Server

Many small changes and improvements with quite big internal code refactors. Previously we had many different Golang packages compiled into Choria Server now we have a mono repo.

A major addition is that you can now view all configuration settings and their values on the CLI, read [our blog post for more details](https://choria.io/blog/post/2020/02/13/configuration/).

We have done major work on Windows support - it can run as a service, write logs to Windows event log and we have initial packages.

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.13.0...0.14.0), [Release](https://github.com/choria-io/puppet-mcollective/releases/tag/0.14.0)

## Enhancements

 * Support use selectable SSL Ciphers using `plugin.security.cipher_suites` and `plugin.security.ecc_curves`
 * Add basic Windows packages
 * Support running as a Windows service
 * Support logging to Windows Event log
 * Update to CloudEvents 1.0.0
 * Merge `go-confkey`, `go-validator`, `go-puppet`, `go-network-broker` `go-protocol`, `go-security`, `mcorpc-agent-provider`, `go-config`, `go-lifecycle` and `go-srvcache` into `go-choria`
 * Set `PATH` when calling external agents
 * Add `choria tool config` to view configuration paramters and current values
 * Cache transport messages when doing batched requests to improve pkcs11 integration
 * Add Debian Buster support
 * Support enforcing the use of filters on all RPC requests using `plugin.choria.require_client_filter`
 * Expose statistics for NATS Leafnodes
 * Export facts to external agents
 * Various improvements to generated RPC clients
 * Install `choria` binary in `/usr/bin` and not `/usr/sbin`
 * Correctly report insecure builds

## Bug Fixes

 * Improve formatting of node lists at the end of requests
 * Ensure agent filter is added when discovering nodes

## [choria/mcollective version 0.10.4](https://forge.puppet.com/choria/mcollective)

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.10.3...0.10.4), [Release](https://forge.puppet.com/choria/mcollective/0.10.4/readme)

### Enhancements

 * Avoid Ruby specific YAML aliases

## [choria/mcollective_choria version 0.17.2](https://forge.puppet.com/choria/mcollective_choria)

Links: [Changes](https://github.com/choria-io/mcollective-choria/compare/0.17.1...0.17.2), [Release](https://forge.puppet.com/choria/mcollective_choria/0.17.2/readme)

## Bug Fixes

 * Fail gracefully when modulepath is unset
 * Improve remote request signing support
 * Correctly extract the playbook name from facts
 * Place the rpcutil DDL in the correct directory

## [choria/choria version 0.16.0](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.15.0...0.16.0), [Release](https://forge.puppet.com/choria/choria/0.16.0/readme)

## Enhancements

 * Add `choria::playbook_exist` function
 * Support Amazon Linux
 * Allow GPG repo checking behaviour to be configured
 * Allow the `ssldir` setting to be configured by the module
 * New configuration options for auto provisioning support
 * Improve windows support
 * Add `choria::sleep` playbook function
