---
title: "September 2019 Bug Fix"
date: 2019-09-20T09:00:00+01:00
tags: ["releases"]
draft: false
---

Today I have a small few bug fixes to ship, these will affect only people who are experimenting or using our newly announced [External Agents](/docs/development/mcorpc/externalagents/) support, others can safely ignore this.

Thanks to Ben Roberts for his assistance with these releases

<!--more-->

## choria/mcollective version 0.10.1

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.10.0...0.10.1), [Release](https://github.com/choria-io/puppet-mcollective/releases/tag/0.10.1)

### Enhancements

 * Allow some files in plugin modules to be executable

## mcorpc-ruby-support gem version 2.20.7

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.20.6...2.20.7), [Release](https://github.com/choria-io/mcorpc-ruby-support/releases/tag/2.20.7)

### Bug Fixes

 * Ensure a valid version of PDK is available when building packages
 * Ensure external agents are executable
 * Only generate JSON DDls when they do not already exist 
 * Avoid duplicate resources for packaged plugins wrt to JSON DDL files

## choria/mcollective_choria version 0.16.1

Links: [Changes](https://github.com/choria-io/mcollective-choria/compare/0.16.0...0.16.1), [Release](https://github.com/choria-io/mcollective-choria/releases/tag/0.16.1)

### Enhancements

 * Require latest ruby support gem