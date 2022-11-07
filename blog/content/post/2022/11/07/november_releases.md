---
title: "November 2022 Releases"
date: 2022-11-07T00:00:00+01:00
tags: ["releases"]
draft: false
---

It's time for a release of the Choria Server and a few related components.  We are hard at work on upgrading
the core security layer and network protocol but managed to slip a few things in while that large piece of
work is progressing.

The security and protocol work is part of a long term goal to not rely so much on Certificate Authorities.
One of the goals is to make Choria more accessible to non Puppet users.  Choria does not require Puppet but
we only really support a Puppet based distribution for general use. Long term we hope this will not be the case.

For those who pay attention to such thing you might be surprised to find the change set between these releases
is in the order of 12,000 LOC, but this is largely an under the covers refactor of code and new features that
are under feature flags. For the typical Puppet user nothing should be changing in this regard apart from the
new features we'll call out here.

## Distributed Goss Validations

We have had support for running [Goss](https://github.com/aelsabbahy/goss) manifests as health checks for a
while and we're expanding that a bit in this release.

First here's some Hiera data to create a goss manifest and then run that as a health check on some nodes,
this will integrate with prometheus for alerting and more:

```yaml
choria::scout_checks:
  check_vpnhosts:
    builtin: goss
    gossfile: /etc/choria/vpnhosts.yaml

choria::scout_gossfile:
  /etc/choria/vpnhosts.yaml:
    addr:
      tcp://10.1.1.1:80:
        reachable: true
        timeout: 500
      tcp://10.1.2.1:80:
        reachable: true
        timeout: 500
```

Goss is great for automated testing in CI and Monitoring but we think it's underappreciated as a debugging
tool.

Lets say you are working an outage of a service, you know the service spans many compute nodes and is made
up of many components. What you want to do is an immediate, deep, health check of the entire service, deeper
than an individual health check tend to do.

```nohighlight
$ choria scout validate /etc/service/goss.yaml -W service=acme
Discovering nodes .... 10

10 / 10    0s [==========================================================] 100%

example.net: Count: 25, Failed: 1, Duration: 0.549s

X Addr: tcp://10.1.2.1:80: reachable:
Expected
bool: false
	to equal
bool: true

Nodes: 10, Failed: 1, Skipped: 0, Success: 250, Duration: 1.251s
```

Here we ran the `/etc/service/goss.yaml` manifest on all machines offering the `acme` service, it found
10 nodes and noted that one of the machines port 80 isn't reachable. It ran 250 checks as this is quite
a small goss file, I have manifests with 100s of resources that execute in sub 1 second.

With the abilities we are adding today you can store a Goss manifest on each machine and then invoke that
in an ad-hoc manner for immediate feedback. You could store different sets of check in the manifest for
different kinds of node - database, processors, API servers etc - and when invoking the check the right deep
validate will be done for the kind of machine.

You can mix in variables either from your shell or from the individual nodes and more.

Manifests can be on the individual nodes as described here or sent from your local shell for truly adhoc inspections.
Manifests sent from local shells can mix in per-node variables to cater for node specific differences and more.

To facilitate sharing between teams you can also store manifests in our Key-Value store, reliably stored in
Choria Streams, so that any team member can access the same set of manifests without any shell setup.

It's early days but I am quite excited for the possibilities here and will look to expand this in time with
more features both client and server side.

## Removal of the Security Cache

We have had a concept of a security cache, here we would store public keys of known users and should the key
change the cache would deny changes. Initially we thought this would be an extra layer of security but it was
largely just a huge UX failure. It made individuals using multiple machines really awkward, resulting in keys
being copied between machines. Most users disabled this feature.

We have entirely removed this concept in this release as it was not useful.

Special thanks to Romain Tartière, Vincent Janelle and Alexander Olofsson for his contributions to this release!

<!--more-->
## [Choria Server version 0.26.2](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.26.1...v0.26.2), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.26.2)

## Enhancements

* Remove the concept of a cache from the security subsystem and other refactors
* Support go 1.18 as minimum version, support go 1.19
* Improve processing of lifecycle events by implementing Stringer for event types
* Work around breaking changes in NATS Server
* Own implementation of the Streams based Governor
* Speed up leader elections
* Restore the ability for provisioners to version update Choria in-place
* Allow direct get to be configured for KV
* Render all tables using UTF-8, remove old table dependency
* Allow RPC clients to supply a goss manifest to execute on the network, from file or KV bucket
* Add the new choria scout validate command that acts as a goss frontend
* Add the delegation property to client JWTs
* Adds an experimental choria tool protocol command that can live view Choria traffic
* Upgrade to a faster and more modern JSON schema validator
* Additional JWT permissions that should be set to allow fleet management access
* Support ed25519 keys for signing JWT tokens
* Allow additional publish and subscribe subjects to be added to client tokens

## Bug Fixes

* Improve flag handling for the rpc builder command
* Do not read config or setup security framework for election file check
* Set up the embedded NATS CLI using the correct inbox prefix
* Improve performance of the optional machines watchers
* Fix building packages for armel
* Avoid some blocking writes in autonomous agent startup, internal efficiency only
* Correctly detect empty filters that might have resulted in unexpected replies
* Fix inventory groups in inventory files, they now work with all agents
* Improve the error handling in choria tool status when the status file does not exist

## [choria/choria version 0.28.4](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.28.3...0.28.4), [Release](https://forge.puppet.com/choria/mcollective_choria/0.28.4/readme)

* Upgrade dependencies


## [choria/mcollective_agent_package version 5.4.0](https://forge.puppet.com/choria/mcollective_agent_package)

Links: [Changes](https://github.com/choria-plugins/package-agent/compare/5.3.0...5.4.0), [Release](https://forge.puppet.com/choria/mcollective_agent_package/5.4.0/readme)

### Bug Fixes

* Run the purge action for uninstalled packages

## [choria-mcorpc-support gem version 2.26.1](https://rubygems.org/gems/choria-mcorpc-support)

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.26.0...2.26.1), [Release](https://rubygems.org/gems/choria-mcorpc-support/versions/2.26.1)

* Report failure with exit code in mco tasks

## [choria/mcollective_choria version 0.21.4](https://forge.puppet.com/choria/mcollective_choria)

Links: [Changes](https://github.com/choria-io/mcollective-choria/compare/0.21.3...0.21.4), [Release](https://forge.puppet.com/choria/mcollective_choria/0.21.4/readme)

### Bug Fixes

* Update DDL files and dependencies
