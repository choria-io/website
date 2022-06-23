---
title: "June 2022 Releases"
date: 2022-06-23T00:00:00+01:00
tags: ["releases"]
draft: false
---

It's time for our next release and this one is packed full of goodies after quite a long development period.
While our previous release was more about internal plumbing this one brings exciting user visible quality of
life improvements and significant new features.

## App Builder

We recently introduced [App Builder](https://choria.io/blog/post/2022/05/25/app_builder/), a operations tool
for building custom CLI commands that wrap many tools into one single app.

In Choria we extend the App Builder to have the ability to interact with Key-Value Stores, initiate RPC requests
and perform discovery.

To use this simply change your app symlink from `appbuilder` to `choria`.

See the [App Builder Documentation](https://choria-io.github.io/appbuilder/) for full reference of the extensions Choria adds.

## `choria` Command UX

We forked the Kingpin CLI parser into a Choria project called [Fisk](https://github.com/choria-io/fisk) that
extends Kingpin in various ways and introduce some breaking changes. We now use Fisk in all our tools.

As part of moving to Fisk the help output from `choria` has been changed significantly to be shorter and easier
to navigate with less superfluous information on every page.

We also add a new `cheat` feature to the CLI that gives you access to cheat sheet style help, here's an example:

```nohighlight
$ choria cheat req
# further documentation https://choria.io/docs/concepts/cli/

# request the status of a service
choria req service status service=httpd

...
```

Run `choria cheat` for a list of available cheats, contributions to these would be greatly appreciated!

## Improved Testing

We now have full integration testing where real running clusters are built and real network round trips are
tested, this is a significant step forward in achieving reliability in the long run.

We also now do daily tests of installing Choria using Puppet on every support Linux Distro.

## Operating Systems

We now publish packages for EL9 and Ubuntu 22.04. Debian packages are tagged by their distribution name to help with mirroring.

We removed Ubuntu Xenial, Debian Stretch and EL6.

## Contributors

Special thanks to Jonathan Matthews, Vincent Janelle, Nicolas Le Gaillart, Lena Schneider  and Romain Tartière for their contributions.

<!--more-->
## [Choria Server version 0.26.0](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.25.1...v0.26.0), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.26.0)

## Removals

 * The Anonymous TLS mode introduced [here](https://choria.io/blog/post/2020/09/13/aaa_improvements/) has been removed in favor for recent JWT enhancements
 * Remove the Provisioner agent `release_update` action that was never used
 * Remove obsolete operating system distributions - EL6, Xenial and Stretch

## Enhancements

 * Debian packages are distro tagged, Ubuntu 22.04 LTS supported but not published due to compatability issues
 * El9 is supported, EL6 removed
 * KV Watcher will now template parse Keys
 * Exec Watcher can now do an initial splayed run before starting schedules
 * Provisioner JWT can have extended details added to it for site specific information
 * UX improvements to --help
 * Cheat Sheet style help via `choria cheat`
 * Client JWT has a new permission that allow access to the system account, system account does not require verified TLS
 * Adds the `choria kv create` and `choria kv update` commands
 * Use `fisk` for the CLI parsing
 * Support Subject Mappings within Choria Broker
 * Embed the `appbuilder` system
 * Reply filters have a new `semver` function
 * Expand the `inventory` registration payload to include version, hash and auto agent information
 * Allow slow TTLs for leader elections
 * Improve reliability of clean shutdowns
 * Reject agents without a name or too small timeout
 * Support skipping system stream management
 * UX improvements for `choria kv`
 * When using the embedded `nats` cli allow a custom Choria configuration to be set
 * Adds full end to end integration testing
 * Improve logging during initial connection establishment
 * Switch to go 1.18
 * Redact some passwords when logging

## Bug Fixes

 * Prevent client permissions from being set on servers, only possible by using the broker as a library
 * Improve validity checks in JWT token caller id
 * Typo fixes in generated clients
 * Work around breaking change in nats.go related to KV access
 * Use correct credentials when running `choria broker server check jetstream`
 * Use correct credentials when running `choria broker server check kv`
 * Improve hostname validation checks in `flatfile` discovery

## [choria/choria version 0.28.1](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.27.3...0.28.1), [Release](https://forge.puppet.com/choria/mcollective_choria/0.28.1/readme)

 * Support Linux Mint and EL9
 * Support distro code tagged deb packages

## [choria/mcollective version 0.14.2](https://forge.puppet.com/choria/mcollective)

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.14.1...0.14.2), [Release](https://forge.puppet.com/choria/mcollective/0.14.2/readme)

 * Support EL9
