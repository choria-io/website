+++
title = "Choria Server"
toc = true
weight = 260
+++

The *Choria Server* new Orchestration Agent that has a `mcollective` compatibility layer and replaces the old *mcollectived* daemon.  This new daemon is written in Golang and it's very fast, light-weight and embeddable. It provides many new features that was never in MCollective.

At present Choria Server is the only supported server, *mcollectived* is no longer supported in any version.

## Status

The aim of the MCollective compatibility layer is to do the bulk of things that the old *mcollectived* did, it might do some things a bit different and downgrade some capabilities, but it is hoped to be a smooth path forward.  Current list of shortcomings / issues are below, please get in touch should you find any more:

  * Compound Filters used in discovery - those with _-S_ - are supported using a new language, no data plugin support yet
  * To use an agent with this daemon you have to repackage it using the latest _mco plugin package_. This ensures that the Ruby language DDL files are translated into portable JSON
  * Agents like the ones that load Puppet - service, package, puppet - are a bit slower by around 1 second per invocation due to _require "puppet"_ being very slow.
  * If you rely on registration in the *mcollectived* you will need to migrate that to our newer format which is much more flexible and scalable (but not yet documented)
  * If you had custom agents and clients that send data types other than JSON primitive data types they will stop working

Thanks to not keeping the whole Puppet in memory it is a lot lighter on your environment:

```
root     28261  0.0  0.1 427508 11512 ?        Ssl  08:52   0:03 /usr/sbin/choria server
root      1553  0.1  1.7 1308192 71964 ?       Sl   06:50   0:07 /opt/puppetlabs/puppet/bin/ruby /opt/puppetlabs/puppet/bin/mcollectived
```

## JSON Pure

YAML on the network has a number of problems and in particular how old MCollective used YAML:

  * It was using Ruby specific data types for every message (Symbols)
  * It has a massive scope for security problems.  We have luckily found very few actionable YAML attacks in MCollective but it's a ever present concern
  * Cross language portability problems

As shown above in the status list there's a huge change happening here - the whole network protocol is now JSON pure.  This is a big big change in behaviour, I've tried to mitigate a lot of this and people who wrote agents as per the official guides will hopefully not notice problems. Further mitigation is needed and I will do those soon once I can be a bit more aggressive in the kinds of changes I can make to the MCollective code. This is a first step towards that.

If however you have your own custom agents and clients/applications I urge you to carefully test them in development under this new mode of operation.

## Configuration

To configure the new daemon you do everything the basic getting started guide shows you. You have to run at least these module versions to ensure the JSON DDL files exist (though not all the actual modules are needed of course):

And you need to change some data:

On all your nodes where you wish to run the new service:

```yaml
choria::server: true
choria::manage_package_repo: true
```

On all nodes including those that are pure MCollective and your clients:

```yaml
mcollective_choria::config:
  security.serializer: "json"
```

The server has lots of configuration options which can be specified by the `choria::server_config` parameter. A full example of the configuration looks like this:

```yaml
classesfile: "/opt/puppetlabs/puppet/cache/state/classes.txt"
  rpcaudit: 1
  plugin.rpcaudit.logfile: "/var/log/choria-audit.log"
  plugin.yaml: "/etc/puppetlabs/mcollective/generated-facts.yaml"
  plugin.choria.agent_provider.mcorpc.agent_shim: "/usr/bin/choria_mcollective_agent_compat.rb"
  plugin.choria.agent_provider.mcorpc.config: "/etc/puppetlabs/mcollective/choria-shim.cfg"
  plugin.choria.agent_provider.mcorpc.libdir: "/opt/puppetlabs/mcollective/plugins"
  plugin.choria.middleware_hosts: "nats1.example.net:4222"
```

For a full list of configuration options, it's best to examine the main choria server config struct defined [here](https://github.com/choria-io/go-config/blob/master/config.go) and the choria plugin struct defined [here](https://github.com/choria-io/go-config/blob/master/choria.go).

At this point you will be running the new Choria daemon. You can confirm this with *mco rpc choria_util info* and you'll see the versions - of course *ps* will also show you.

The MCollective subsystem will still log to your normal *mcollective.log* and auditing will also go to the log configured and previously used for mcollective, formats of those would not have changed yet.
