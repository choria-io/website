---
title: "Choria Hierarchical Data"
date: 2025-11-25T00:00:00+01:00
tags: ["hiera", "releases"]
draft: false
---

As most are aware, I created the widely used Hiera system in Puppet. I [introduced it in 2011](https://www.devco.net/archives/2011/06/05/hiera_a_pluggable_hierarchical_data_store.php), and it has since become essentially the only way to use Puppet in any meaningful fashion. Given its widespread adoption, I donated the code to Puppet, and it became integrated with Puppet core.

Unfortunately, during this integration we lost some key values—the command line and the ability to use it in scripts and elsewhere.

Meanwhile, our world is changing, and we are ever more focussing on small, single purpose compute. I intend to create a new kind of Configuration Management system that is focussed on the small, single, purpose needs. Filling the gap that has always existed - how to manage our applications rather than systems, an area Puppet has always been weak at.

There is then still the need for hierarchical data and given that I have the flexibility to start completely fresh I am at best taking some inspiration from Hiera.

So today I'll introduce a new tool called Choria Hierarchical Data - current code name `tinyhiera` but that might change.

Read on for the details

<!--more-->

The big difference here is that this new system supports just one data file, you express a data structure that is the complete outcome of the query and then configure in the same file how to override and extend that data file.

Let's look at it, here I'll show the different sections, in reality it's just one file.

This is the data we wish to manage:

```yaml
data:
    log_level: INFO
    packages:
        - httpd
    web:
        listen_port: 80
        tls: false
```

Lets define the hierarchy for overriding this data; this should be familiar to Puppet users:

```yaml
---
hierarchy:
    merge: deep # or first
    order:
        - env:{{ lookup('env') }}
        - role:{{ lookup('role') }}
        - host:{{ lookup('hostname') }}
```

And finally, we show the overrides, here we just specify how the data will be extended:

```yaml
overrides:
    env:prod:
        log_level: WARN

    role:web:
        packages:
            - nginx

        web:
            listen_port: 443
            tls: true

    host:web01:
        log_level: TRACE
        web:
            listen_port: 8080
```

We can now call our CLI tool to resolve this entire structure.

We query the data setting the `role` fact to `web`, the data from the `role:web` section is merged into the data from the `data` section:

```nohighlight
$ tinyhiera parse data.yaml role=web
{
  "log_level": "INFO",
  "packages": [
    "ca-certificates",
    "nginx"
  ],
  "web": {
    "listen_port": 443,
    "tls": true
  }
}
$ tinyhiera parse data.yaml role=web --query web.listen_port
443
```

## Features and Differences

This is the basic feature, similar to what you might have used in Puppet except differing in a few areas:

 * It's all one file and one data structure, it's not key-lookup orientated. The entire outcome is rendered in one query
 * Expressions used in overrides and data lookups are a full query language called [expr-lang](https://expr-lang.org), this means we can call many functions, generate data, derive data and easily extend this over time
 * The data is typed, if your facts have rich data the results can also be rich data, if `value` is a integer, array or map the result will be the same type `x="{{ lookup("value") }}"`
 * The CLI has built in system facts, can take facts on the CLI, read them from JSON and YAML files, parse your environment variables as facts or all at the same time
 * The CLI can emit data as environment variables for use in scripts
 * There is a Golang library to use it in your own code

## Environment Variable Output

We want this to be usable in scripts, the easiest might be dig into the data and just get a single value:

```nohighlight
PORT=$(tinyhiera parse data.yaml role=web --query web.listen_port)
```

But this involves many calls to the command, we can instead emit all the data as variables.

```nohighlight
$ tinyhiera parse data.yaml role=web --env
HIERA_LOG_LEVEL=INFO
```

You can see the data is emitted as environment variables that you can just source into your shell script, obviously in this use case it benefits to have flat data.

## Built-in facts

We have many ways to get facts into it such as files, environment variables, JSON and YAML files. The CLI also includes it's own mini fact source called `system` which will return facts about the system it is running on.

To view the facts fully resolved, we can run the following, I am removing much of the details here but you can see this offers enough to make most required decisions:

```nohighlight
$ tinyhiera facts --system-facts
{
  "host": {
    "info": {
      "hostname": "grime.local",
      "uptime": 1843949,
      "bootTime": 1762258234,
      "procs": 711,
      "os": "darwin",
      "platform": "darwin",
      "platformFamily": "Standalone Workstation",
      "platformVersion": "26.1",
      "kernelVersion": "25.1.0",
      "kernelArch": "arm64"
    }
  },
  "memory": {
    "swap": { },
    "virtual": { }
  },
  "network": {
    "interfaces": [ ]
  },
  "partition": {
    "partitions": [ ],
    "usage": [ ]
  }
}
```

## Long-term view

### Autonomous Agents

The goal here is to incorporate this into various other places, first we pull it into a [Choria Autonomous Agents](https://choria.io/docs/autoagents/), these agents own the full lifecycle of an application:

 * Deploy dependencies
 * Configure the application
 * Run the application
 * Monitor the application
 * Restart the application on failure
 * Orchestrate rolling upgrades
 * Present APIs for interacting with the management layer

Agents can fetch data from a Key-Value store, but it's kind of all the same data. With Hiera integrated into the KV feature, we get:

```yaml
  - name: watch_tag
    type: kv
    interval: 10s
    success_transition: regional_update
    state_match: [RUN, RESTART]
    properties:
      bucket: NATS
      key: config
      hiera_config: true
```

Here the autonomous agent will check KV every 10 seconds for data, fully resolve it using Hiera and save the resulting data into the Autonomous Agent internal data store. If the KV data chanfes or the referenced facts change to the extent that the data change, the machine will update the stored data and fire the `regional_update` event.

This way we can create role-orientated specific data all in the same Key-Value key.

### Configuration Management

I want to create a new CM system that takes the model we are used to in Puppet but brings it to scripts and reusable APIs.

The script use case would essentially be, these commands would be idempotent which would hugely improve the ability for simple scripts to be safe to run many times:

```nohighlight
$ marionette package ensure zsh --version 1.2.3
$ marionette service ensure ssh-server --enable --running
$ marionette service info httpd
```

This should be familiar to Puppet users, we're basically pulling the resources and RAL into standalone commands.  The commands will be fully idempotent like Puppet and support multiple Operating Systems.

For integration with other languages, you can also deal with JSON-in-JSON-out style:

```nohighlight
# Can be driven by JSON in and returns JSON out instead
$ echo '{"name":"zsh", "version":"1.2.3"}'|marionette package apply
{
....
}
```

I do not want to create something huge like Puppet, but just basically enough that can enable the package-config-service trio pattern, ultimately a manifest would look like this:

```yaml
cat <<EOF>manifest.yaml
data:
    resources:
        - package:
            name: myapp
            ensure: present
        - service:
            name: myapp
            ensure: running
            enabled: true

hierarchy:
    merge: deep
    order:
        - platform:{{ lookup('host.info.platform') }}
        - hostname:{{ lookup('hhost.info.hostname') }}

overrides:
    platform:darwin
        resources:
            - package:
                ensure: MyApp-1.2.3.dmg
EOF

$ marionette apply manifest.json --system-facts
```

## Availability and Status

You can get the command on [GitHub choria-io/tinyhiera](https://github.com/choria-io/tinyhiera), we are pretty far along but expect some more breaking changes including a potential name change.