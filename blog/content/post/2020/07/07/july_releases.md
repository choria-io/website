---
title: "July 2020 Releases"
date: 2020-07-07T09:00:00+01:00
tags: ["releases"]
draft: false
---

We have a number of releases to announce today, the focus is general quality of life improvements in addition to 
the features to support out larger Choria Server release that included our [announcement of Choria Scout](https://choria.io/blog/post/2020/07/02/choria_scout/).

With these releases you can create Scout checks on your machines using:

```puppet
choria::scout_check{"check_typhon":
    plugin            => "/usr/lib64/nagios/plugins/check_procs",
    arguments         => '-C typhon -c {{ o "warn" 1 }}:{{ o "crit" 1 }}',
    remediate_command => "service typhon restart",
}
```

In addition to this we have fixed `mco puppet runall` when using Choria Server, I know quite a few people have wanted to
see the return of this utility.

Thanks to Romain Tartière for contributions to these releases.

<!--more-->
## [choria/mcollective_agent_puppet version 2.3.3](https://forge.puppet.com/choria/mcollective_agent_puppet)

Links: [Changes](https://github.com/choria-plugins/puppet-agent/compare/2.3.2...2.3.3), [Release](https://forge.puppet.com/choria/mcollective_agent_puppet/2.3.3/readme)

### Enhancements

 * Support runall on Choria Server

## [choria/mcollective_agent_package version 5.3.0](https://forge.puppet.com/choria/mcollective_agent_package)

Links: [Changes](https://github.com/choria-plugins/package-agent/compare/5.2.0...5.3.0), [Release](https://forge.puppet.com/choria/mcollective_agent_package/5.3.0/readme)

### Enhancements

 * Support FreeBSD
 * Adds a `refresh` action to update available packages
 
### Bug Fixes

 * Improve handling of operating systems that does not report architecture in packages
 * Improve handling absent and purged packages

## [choria/mcollective_agent_nettest version 4.0.3](https://forge.puppet.com/choria/mcollective_agent_nettest)

Links: [Changes](https://github.com/choria-plugins/nettest-agent/compare/4.0.2...4.0.3), [Release](https://forge.puppet.com/choria/mcollective_agent_nettest/4.0.3/readme)

### Bug Fixes

 * * Improve dependencies on the net-ping gem

## [choria-mcorpc-support gem version 2.21.1](https://rubygems.org/gems/choria-mcorpc-support)

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.21.0...2.21.1), [Release](https://rubygems.org/gems/choria-mcorpc-support/versions/2.21.1)

## Enhancements

 * Fix facts application when querying non-string facts

## [choria/mcollective_choria version 0.17.3](https://forge.puppet.com/choria/mcollective_choria)

Links: [Changes](https://github.com/choria-io/mcollective-choria/compare/0.17.2...0.17.3), [Release](https://forge.puppet.com/choria/mcollective_choria/0.17.3/readme)

### Bug Fixes

 * Print CSR fingerprint in `request_cert` application
