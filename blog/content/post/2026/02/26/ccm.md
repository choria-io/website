---
title: "Choria Configuration Manager"
date: 2026-02-26T00:00:00+01:00
tags: ["hiera", "ccm", "releases"]
draft: false
---

I've often wanted to have Configuration Management as a library - a set of tools I can use to experiment with features in my
Autonomous Agents to build new kinds of highly reactive and distributed configuration management.

After spending a lot of time looking at the various options I finally caved and started working on this as a Choria project.

At first, as stated, focussed on the library aspect of the project but I soon realized that I have a long standing need that existing
CM projects did not address.

That need is about my home lab, dev setups, throw away machines and places where I want to experiment with software. I want Configuration Management
that's tailor made fot these unpredictable environments where every node is not just a pet but a snow flake - completely unique.

So using this library I started a new Configuration Management project called [Choria Configuration Manager](https://choria-ccm.dev/) - or simply CCM. The focus is on bringing
Configuration Management to places it has not been before:

 * Ad-hoc on the CLI
 * In shell scripts
 * In YAML manifests
 * From any programming language that can interpret JSON
 * In Choria Autonomous Agents
 * As a library

It's true we've had many of these before but CCM is focused on bringing high quality Configuration Management to all these places.

That means Idempotency, Desired State, Noop mode, Transactional behaviors (notify and subscribe), Hierarchical Data, System Facts etc.

Of course ultimately the goal is to bring Configuration Management into Autonomous Agents where it will bring the capabilities of CCM 
to very large scale deployments. But that's the beauty of CM as a library, we can use it to solve many kinds of problems with consistency
and compatibility between different worlds.

Read on for the details

<!--more-->

Let's consider a Puppet Manifest like this:

```puppet
package{"httpd":
  ensure => present,
}

file{"/etc/httpd/conf.d/listen.conf":
  ensure => present,
  content => "Listen 8080\n",
}

service{"httpd":
  ensure    => running,
  enable    => true,
  subscribe => File["/etc/httpd/conf.d/listen.conf"]
}
```

To execute this requires 100s of MB of Ruby Language code and comes with many seconds of overhead just starting Ruby etc.

This is fine in a world where interactivity does not matter - long running agents managing systems in the background. But in
systems where ad-hoc interaction is important, the costs is prohibitive.  Costs being both the overhead of installing everything
but also just the time and general annoyance of using Puppet interactively – not to mentioning learning a new language. This is
not a slight on Puppet, this area is just not where it's built to be used so if anything it's a case of wrong tool for the job.

What if instead you could write a shell script to achieve true configuration management? That means:

 * **Desired State** - You say what you want, the system works out the steps needed to get you there.
 * **Idempotent** - The system will only make changes when needed, not every time you run it. You can run shell scripts many times safely.
 * **Fast** - The overhead that the tool brings is negligent, one static binary with no other dependencies that when the system is stable completes this take in sub **200ms**.
 * **Portable** - With access to system facts and hierarchical data you can run the same script on any system.
 * **Easy to adopt** - You do not need to learn a new language, just use any shell you want.
 * **Supports Hierarchical Data** - Loads data from files, network etc and resolves it hierarchically like Hiera does with no extra dependencies.
 * **Supports Transactional Behavior** - Changes in one resource can result in corrective behaviours in others (subscribe, notify etc)

CCM let's you do all this and more, here is the above Puppet Manifest expressed in CCM shell scripts:

```bash
#!/bin/bash

eval $(ccm session new)
ccm ensure package httpd
ccm ensure file /etc/httpd/conf.d/listen.conf content="Listen 8080"
ccm ensure service httpd --subscribe file#/etc/httpd/conf.d/listen.conf
ccm session report --remove
```

Of course not everyone likes Shell scripts, we also have manifests. Here is a more complete example that includes Hierarchical data, OS overrides and more. However even the shell scripts have these features!

```yaml
data:
  package_name: httpd
  service_name: httpd

ccm:
  resources:
    - package:
        - "{{ Data.package_name }}":
            ensure: present
    - file: 
        - "/etc/httpd/conf.d/listen.conf":
            ensure: present
            content: |
                Listen 8080
    - service:
        - "{{ Data.service_name }}":
            ensure: running
            enable: true
            subscribe: 
              - file#/etc/httpd/conf.d/listen.conf

hierarchy:
  order:
    - os:{{ lookup('facts.host.info.platformFamily') }}

overrides:
  os:debian:
    package_name: apache2
    service_name: apache2
```

There is much more to the explore, I've introduced the project this year at Configuration Management Camp in Ghent and have re-recorded the talk I gave, you can see it [here on Youtube](https://www.youtube.com/watch?v=Ka5MnTxjJiE).

## Status

The project is quite new, only a few months.  We're focussing on the core need and not looking to expand to 100s of resource types - package-config-service is the sweet spot.

We have Package, File, Service, Archive, Scaffold and Exec types supporting Debian and RedHat based systems.

You can get it at [choria-cm.dev](https://choria-cm.dev/).  Thanks to LLMs I've also been able to deliver a graphic manafist editor using a modern web stack
You can find if [CCM Studio](https://studio.choria-cm.dev/).
