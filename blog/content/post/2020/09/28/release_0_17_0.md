---
title: "Choria Server 0.17.0"
date: 2020-09-28T09:00:00+01:00
tags: ["releases"]
draft: false
---

Today we have quite a bumper release with significant updates for [Choria Scout](https://choria.io/docs/scout/) and
the first step in [improvements for AAA Service managed clients](https://choria.io/blog/post/2020/09/13/aaa_improvements/).

We added numerous Choria Scout CLI tools - `choria scout status`, `choria scout trigger`, `choria scout maintenance`
and `choria scout resume`.  These allow you to manage a fleet of Choria nodes that are performing Scout checks.

```nohighlight
$ choria scout status dev1.example.net
+-----------------------+-------+------------+-------------------------------+
| NAME                  | STATE | LAST CHECK | HISTORY                       |
+-----------------------+-------+------------+-------------------------------+
| mailq                 | OK    | 1m20s      | OK OK OK OK                   |
| ntp_peer              | OK    | 1m32s      | OK OK OK OK OK OK OK OK OK OK |
| pki                   | OK    | 2m28s      | OK OK OK OK OK OK OK OK OK OK |
| puppet_failures       | OK    | 2m3s       | OK OK OK OK WA WA CR CR OK OK |
| puppet_run            | OK    | 24s        | OK OK OK                      |
| swap                  | OK    | 4m23s      | OK OK OK OK OK OK OK          |
| zombieprocs           | OK    | 2m23s      | OK OK OK OK OK OK OK OK OK OK |
| goss                  | OK    | 3m12s      | OK OK OK                      |
| heartbeat             | OK    | 57s        | OK OK OK OK OK OK OK OK OK OK |
+-----------------------+-------+------------+-------------------------------+
```

The `choria req` utility got a new `--table` format option and all the result rendering code got extracted into a 
reusable package.

```nohighlight
[rip@dev1]% choria req package status package=zsh --table
Discovering nodes .... 2

2 / 2    0s [====================================================================] 100%

+------------------+--------+------------------+-------+------+--------+----------+------------+---------+
| SENDER           | ARCH   | ENSURE           | EPOCH | NAME | OUTPUT | PROVIDER | RELEASE    | VERSION |
+------------------+--------+------------------+-------+------+--------+----------+------------+---------+
| dev2.example.net | x86_64 | 5.0.2-34.el7_8.2 | 0     | zsh  |        | yum      | 34.el7_8.2 | 5.0.2   |
| dev1.example.net | x86_64 | 5.0.2-34.el7_8.2 | 0     | zsh  |        | yum      | 34.el7_8.2 | 5.0.2   |
+------------------+--------+------------------+-------+------+--------+----------+------------+---------+
```

We improved generated Go clients significantly by allowing them to have typical progress bars, `choria req` like result
formatting, result parsing helpers, improved logging and faster discovery.  These features are show cased in the new 
`choria scout` commands that are built entirely by using abilities of the generated clients. We also significantly 
simplified the code for `choria req` by using the same features.
   
Shout out to Romain Tartière and Mike Newton for their contribution

<!--more-->
## [Choria Server version 0.17.0](https://github.com/choria-io/go-choria)

Links: [Changes](https://github.com/choria-io/go-choria/compare/v0.16.0...v0.17.0), [Release](https://github.com/choria-io/go-choria/releases/tag/v0.17.0)

 * Add a generic shell completion helper and support ZSH completion                                       
 * Support NATS Leafnodes to extend the Choria Broker in a TLS free way specifically usable by AAA clients
 * Scout checks can have annotations that are published in events                                         
 * Add `choria scout maintenance` and `choria scout resume` commands                                      
 * Add a `choria scout trigger` command that triggers an immediate check and associated events            
 * Generated clients can now set a progress bar                                                           
 * Prevent int overflow in time fields in some Scout events                                               
 * Add a `--table` option to `choria req` and a new formatter in generated clients                        
 * Add a `choria scout status` command that can show all checks on a node                                 
 * Improve the history presented in Scout events                                                          
 * Remove the concept of a site wide Gossfile                                                             
 * Allow multiple Gossfiles and multiple Goss checks                                                      

## [choria-mcorpc-support version 0.22.0](https://rubygems.org/gems/choria-mcorpc-support)

Links: [Changes](https://github.com/choria-io/mcorpc-ruby-support/compare/2.21.1...2.22.0), [Release](https://rubygems.org/gems/choria-mcorpc-support/versions/2.22.0)

 * Support FreeBSD

## [choria/mcollective version 0.11.0](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-mcollective/compare/0.10.5...0.11.0), [Release](https://forge.puppet.com/choria/mcollective/0.11.0/changelog)

 * Add dash to the Mcollective::Collectives pattern

## [choria/mcollective_choria version 0.19.0](https://forge.puppet.com/choria/mcollective_choria)

Links: [Changes](https://github.com/choria-io/mcollective-choria/compare/0.18.0...0.19.0), [Release](https://forge.puppet.com/choria/mcollective_choria/0.19.0/readme)

 * Support JWT based anonymous broker TLS

## [choria/choria version 0.19.0](https://forge.puppet.com/choria/choria)

Links: [Changes](https://github.com/choria-io/puppet-choria/compare/0.18.0...0.19.0), [Release](https://forge.puppet.com/choria/choria/0.19.0/readme)

 * Support multiple Gossfiles
 * On resuming a Scout check move to `unknown` not `force`
 * Support Scout check annotations
