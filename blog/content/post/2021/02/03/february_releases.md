---
title: "February 2021 Releases"
date: 2021-02-03T00:00:00+01:00
tags: ["releases"]
draft: false
---

Hot on the heels of our January release we have a few small bug fixes to the previous release, and a number of very significant improvements to the discovery and configuration subsystems.

This is again a big release, and we suggest you do careful testing of your client applications in your testing environments after reading the `Upgrade Notes` in this post.

The focus of this release has been around Discovery and Configuration, we'll let the planned module changes bake a bit longer to ensure we're 100% stable where we are now before we undertake the next big change. Discovery features no fewer than 3 new discovery methods, we have the start of Data Providers in Compound Filters and exciting new project based configuration, read the full post for details.

Special thanks to Vincent Janelle, Romain Tartière and Ben Roberts for their contributions in this release.

<!--more-->
## Discovery Improvements

The full discovery feature set have been documented at [https://choria.io/docs/concepts/discovery/](https://choria.io/docs/concepts/discovery/), below is a list of improvements in this release:

 * New `external` discovery method allowing any programming language to be used to extend the discovery subsystem. [Go](https://github.com/choria-io/go-external) and [Python](https://github.com/optiz0r/py-mco-agent) supported with helpers.
 * New `inventory` discovery method to discover from a local state cache and supports prepared queries and compound filters
 * New `flatfile` method supporting JSON, YAML, Text Files and STDIN as sources. For YAML and JSON supports using GJSON to query nested structures.
 * Restore the `--nodes` flag, can now read any of the flatfile sources
 * Request chaining and filtering in the `choria req` command: `choria req package status package=php --filter-replies 'ok() && data("ensure")=="5.3.3-49.el6"' -j| choria req service restart service=httpd`
 * Support passing options to discovery methods using `--do`
 * When supplying `-I` filters that are not regular expressions, pql queries or inventory groups avoid discovery all together. Avoids crashing PuppetDB.
 * `choria discover` can now be used by scripts to perform discovery using the `--silent` and `--json` flags
 * Support data providers in the `-S` compound queries. Ship `choria`, `scout` and `config_item` data providers
 * Support all new discovery features in Ruby by invoking the `choria discover` command for discovery

This is a huge list of improvements, discovery in Choria is now better than it's ever been and our `external` discovery method is the first offline method to support compound queries. We'll post followup blog posts with details on some of these, meanwhile the new documentation covers everything.

### Inventory Discovery

I want to focus a bit on the `inventory` discovery method. This method allow you to maintain a YAML or JSON file that holds:

 * Detail about nodes - names, facts, classes, sub collective membership
 * Named node groups - essentially pre-created queries that has a name

Below is an example inventory file with 2 node groups - `web` and `db` - and 2 machines in it.  We can target all webservers using `-I group:web` when this discovery method is active.

Maintaining these files are a bit of a drag, you can see we have a `$schema` though, and many editors will download this schema and help you when editing the file.

If you already have these machines you can use Choria to build and maintain the inventory for you:

```nohighlight
$ choria tool inventory --update --dm mc inventory.yaml -W customer=acme
Discovering nodes .... 2

Wrote 2 nodes to /home/user/projects/web/inventory.yaml
```

Any group definitions that were already in your inventory.yaml will be kept, all the facts, classes etc will be updated in the file - so you don't need to maintain it by hand.

This method also supports compound filters to `choria discovery -S "with("nginx") && with("environment=prod")"` would totally work. This is the first time ever we support offline compound queries. Though, of course, without Data Providers.

While I would not recommend this, I've tested this with a 180 MB inventory JSON file containing 10 000 nodes - it worked fine.  The intended use though is for up to 1000 nodes I think, past that other methods will probably work better.

```yaml
$schema: https://choria.io/schemas/choria/discovery/v1/inventory_file.json
groups:
- name: web
  filter:
    classes:
      - nginx
- name: db
  filter:
    classes:
      - mysql
nodes:
  - name: n1.example.net
    classes:
      - nginx
    facts:
      customer: acme
      environment: prod
    collectives:
      - mcollective
      - customers  
  - name: n1.example.net
    classes:
      - mysql
    facts:
      customer: acme
      environment: prod
    collectives:
      - mcollective
      - customers
```

## Configuration Improvements

In this release Romain Tartière added a great new feature to support project based configuration.  In short, if in a directory `/home/user/projects/web` you create `choria.conf` this file is read and applied over the normal configuration. This allows you to configure custom discovery, SSL, brokers and more on a per project basis.

First, we documented all the details about configuration, what files are queried and how, how to inspect running configuration, the file format and more.  This can e found on our docs site [https://choria.io/docs/configuration/reference/](https://choria.io/docs/configuration/reference/).

The new `choria tool config` improvements gives insight into the running configuration:

```nohighlight
$ cat choria.conf
plugin.choria.discovery.inventory.source = inventory.yaml
default_discovery_method = inventory
$ ls -l 
-rw-rw-r-- 1 rip rip     58 Feb  3 11:17 choria.conf
-rw------- 1 rip rip 660300 Feb  3 11:18 inventory.yaml
$ choria discover -I group:webservers -v
Discovering nodes using the inventory method .... 19
...
Discovered 27 nodes using the inventory method in 0.19 seconds
```

You can see here we have a project specific configuration file `choria.conf` that sets this project to use the new inventory discovery method and has a custom inventory there. We discover the `group:webservers` in the inventory.

You can see your active configuration and what files were involved in resolving that configuration:

```nohighlight
$ choria tool config
Configuration Files:

   User Config: /etc/choria/client.conf
  Loaded Files: /etc/choria/client.conf
                /etc/choria/plugin.d/actionpolicy.cfg
                /etc/choria/plugin.d/choria.cfg
                /etc/choria/plugin.d/login.cfg
                /etc/choria/plugin.d/nrpe.cfg
                /etc/choria/plugin.d/puppet.cfg
                /home/user/work/web/choria.conf

Loaded Configuration:

                                       collectives: mcollective,mt_collective
                          default_discovery_method: inventory
                                          loglevel: warn
                                   main_collective: mcollective
```

## Upgrade Notes

We've done a lot of work on the discovery subsystem in this release, and some of it can simply not be made available to the Ruby client natively. We've therefore taken the decision to delegate all discovery from the Ruby client to the *choria* binary. The Ruby libraries will invoke *choria* and it will do the discovery.  This might seem a bit weird, but it's actually working out well in my experience. The Go clients can discover a 50 000 network in sub 2 seconds, a huge improvement over Ruby, overall this should improve the stability and repeatability of discovery for Ruby in addition to adding new behaviours. On the development side this removes a lot of duplicate code and means its easier for me to deliver a bug free experience.

Going forward we have to support at minimum the *mcorpc-ruby-support* gem version *2.24.1* and Choria Server at *0.20.1*, the Puppet modules will perform the Gem update, On RedHat and Debian/Ubuntu we will bump the Choria version, others will have to configure `choria::version` Hiera data to your version needs, but must be at least *0.20.1*

## [Choria Server version 0.20.2](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.19.0...v0.20.2), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.20.2)

### Enhancements

 * Sort classes tags in discovery command and elsewhere
 * Initial support for Data Providers, add `choria`, `scout`, `config_item` providers
 * Perform identity-only discovery optimization in `broadcast` and `puppetdb` discovery methods
 * Add a `--silent` flag to `choria discover` to improve script integration
 * Support go 1.15 by putting in work around to support Puppet SAN free TLS certificates
 * Add a bash completion script in `choria completion` in addition to current ZSH support
 * Adds a new `inventory` discovery method
 * Improve SRV handling when trying to find PuppetDB host
 * Improve `choria tool config` to show config files and active settings
 * Add project level Choria configuration
 * Allow options to be passed to discovery methods using `--do`
 * Support flatfile discovery from json, yaml, stdin and improve generated clients. Restore the `--nodes` flag
 * Add the `external` discovery method
 * Support request chaining in the req command
 * Restore the `rpcutil#get_config_item` and `rpcutil#get_data` actions

## Bug Fixes

* Improve progress bars on small screens
* Ensure we discover `rpcutil` in the `discover` command, improves PuppetDB integration
* Performance improvements for expr expression handling
* Improve identity handling when running on windows, non root and other situations

## [choria-mcorpc-support gem version 2.24.1](https://rubygems.org/gems/choria-mcorpc-support)

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.23.3...2.24.1), [Release](https://rubygems.org/gems/choria-mcorpc-support/versions/2.24.1)

**WARNING:** To use this version of the Gem you must have *Choria Server version 0.20.1* available locally.

### Enhancements

 * Remove legacy Data plugins
 * Delegate all discovery methods to the choria binary
 * Support project based configuration files

## Bug Fixes

* Improve rendering of packaged plugin README documents

## [choria/mcollective version 0.13.1](https://forge.puppet.com/choria/mcollective)

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.13.0...0.13.1), [Release](https://forge.puppet.com/choria/mcollective/0.13.1/readme)

### Enhancements

 * Support agents without any extension, improving support for packaging `external` agents

## [choria/choria version 0.22.2](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.21.0...0.22.2), [Release](https://forge.puppet.com/choria/choria/0.22.2/readme)

### Enhancements

 * Support `run_as` for playbooks
 * Ensure the `mcollective` class is included when just including `choria`
 * Ensure version `0.20.2` of Choria is installed on RedHat and Debian/Ubuntu machines
