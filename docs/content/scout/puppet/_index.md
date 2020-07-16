+++
title = "Configuration"
weight = 10
+++

## Overview

For users of the Open Source system as delivered using our Puppet modules it's easy to configure Scout today. This guide
will cover the available options and integrations.

## Checks

A Scout Check is a specialized Choria Autonomous Agent using the [Nagios watcher](https://choria.io/docs/autoagents/watcher_reference/#nagios-watcher)
to continuously run checks and perform remediation.

The basic check is a Nagios compatible plugin, meaning the exit code is meaningful:

|Exit Code|State|Description|
|---------|-----|-----------|
|0        |OK   |The plugin was able to check the service and it appeared to be functioning properly|
|1        |WARNING|The plugin was able to check the service, but it appeared to be above some "warning" threshold or did not appear to be working properly|
|2        |CRITICAL|The plugin detected that either the service was not running or it was above some "critical" threshold|
|3        |UNKNOWN|Invalid command line arguments were supplied to the plugin or low-level failures internal to the plugin (such as unable to fork, or open a tcp socket) that prevent it from performing the specified operation. Higher-level errors (such as name resolution errors, socket timeouts, etc) are outside of the control of plugins and should generally NOT be reported as UNKNOWN states.|

Additionally, we attempt to parse well formed Nagios Performance data.

A basic Nagios based check can be configured like this:

```puppet
choria::scout_check{"mailq":
    plugin             => "/usr/lib64/nagios/plugins/check_mailq",
    arguments          => "-M exim -w 5 -c 10",
}
```

This will execute the `check_mailq` command every 5 minutes and report the status.  There's a number of options that 
we'll cover below.

You can set a timeout by passing `plugin_timeout => "10s"`, this is how long the `check_mailq` command is allowed to run,
it defaults to 10 seconds.

The check will be run on an interval configured using `check_interval => "5m"`, it defaults to 5 minutes.

### Hiera

Checks can be added using Hiera by creating data like this:

```yaml
choria::scout_checks:
    heartbeat:
      builtin: heartbeat
      check_interval: 1m

    check_mailq:
      plugin: /usr/lib64/nagios/plugins/check_mailq
      arguments: -M exim -w {{ o "warn" 5 }} -c {{ o "crit" 10 }}
```

### Node specific overrides

In the above Hiera example you see a check like this:

```yaml
check_mailq:
  plugin: /usr/lib64/nagios/plugins/check_mailq
  arguments: -M exim -w {{ o "warn" 5 }} -c {{ o "crit" 10 }}
```

Here the `{{ o "warn" 5 }}` will retrieve a node specific override that defaults to 5 for the warning threshold.

The data can be set in Hiera:

```yaml
 choria::scout_overrides:
   check_mailq:
     warn: 15
     crit: 25
```

The `warn` key will be fretched from a Hash matching the check name.

### Remediation

Remediation is supported by running a supplied command when the check is in a specific state. The remediation command is
run on a specific frequency as long as the state matches, this means remediation will be attempted over and over as long
as the check is in the specified status.

```puppet
choria::scout_check{"mailq":
    plugin             => "/usr/lib64/nagios/plugins/check_mailq",
    arguments          => "-M exim -w 5 -c 10",
    remediate_command  => "/usr/local/bin/runq.sh"
}
```

Here we specify that the `runq.sh` is run when the check is in `CRITICAL` state. Possible states are `UNKNOWN`, `OK`,
`WARNING`, `CRITICAL`.

The states that will trigger remediation can be set using `remediate_states => ["CRITICAL"]`, this is the default setting.

The frequency of remediation attempts can be set using `remediate_interval => "15m"`, 15 minutes is the default.

### Heartbeats

Scout supports publishing a heartbeat using a special built-in plugin type, this can be configured as here:

```puppet
choria::scout_check{"heartbeat":
    builtin        => "heartbeat",
    check_interval => "1m"
}
```

### Goss

Scout supports running [Goss](https://github.com/aelsabbahy/goss) validations regularly and treating their outcome as
check states.

```puppet
choria::scout_check{"goss":
    builtin => "goss"
}
```

The gossfile contents can be set via Hiera:

```yaml
choria::scout_gossfile:
  port:
    tcp:22:
      listening: true
      ip:
      - 0.0.0.0
  service:
    sshd:
      enabled: true
      running: true
```

In your gossfile the Overrides data can be used using it's templating via the _{{.Vars}}_.

## Prometheus

Scout can integrate with [Prometheus](https://prometheus.io/) Node Exporter, this requires the `textfile` collector path
to be configured:

```yaml
choria::server_config:
    plugin.choria.prometheus_textfile_directory: /var/lib/node_exporter/textfile
```

If you use the [Vox Pupuli Prometheus Module](https://forge.puppet.com/puppet/prometheus) you need this configuration:

```yaml
prometheus::node_exporter::extra_options: "--collector.textfile.directory=/var/lib/node_exporter/textfile"
```

And then include the `prometheus::node_exporter` class.

For details about checks and alerts please refer to the dedicated Prometheus section.
