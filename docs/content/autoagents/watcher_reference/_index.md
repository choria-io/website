+++
title = "Watcher Reference"
weight = 40
toc = true
+++

{{% notice tip %}}
A [JSON schema](https://choria.io/schemas/choria/machine/v1/manifest.json) describes these files and you can configure some editors to validate the YAML file based on that. The command `choria machine validate` can validate a *machine.yaml* against this schema.
{{% /notice %}}

## Common watcher properties

All watchers share a common set of properties detailed below, watcher specific properties are always in the *properties* key:

|Property          |Required                    |Description|
|------------------|----------------------------|-----------|
|name              |yes|A unique name for the watcher|
|type              |yes|A known type of watcher like *file* or *exec*|
|state_match       |no |A list of state names where this watcher is valid for|
|fail_transition   |   |If set this event fires on failure|
|success_transition|   |If set this event fires on success|
|interval          |   |Runs the watcher every interval, valid intervals are of the form *1s*, *1m*, *1h*|
|announce_interval |   |Announce the current state of the watcher regularly, valid intervals are of the form *1s*, *1m*, *1h*|
|properties        |yes|Watcher specific settings|

## File watcher

The *file* watcher observes a specific file for changes and presence. Today only a basic *mtime* check is done, in time other dimensions like hashes or even *inotify* based observation will be supported.

### Properties

|Property            |Required                            |Description|
|--------------------|------------------------------------|-----------|
|path                |yes|The path to the file to watch relative to the watcher manifest directory|
|gather_initial_state|   |Gathers the initial file mode, stats etc for regular announces but only perform first watch after *interval*|

### Behavior

A file watcher will at *interval* times do an *mtime* check on the file.

If the file is missing a *fail_transition* event fires and an announcement is made.

If the file has changed since the previous run a *success_transition* event fires and an announcement is made. This means the first check would set the initial state after which changes are detected.  You could by setting *gather_initial_state* have the system gather initial file state on startup so the first regular watch would detect a change.

If the file has not changed nothing is published on every check, however a regular state announce can be done by setting *announce_interval*.

## Exec watcher

The *exec* watcher supports running shell commands, it has a very basic exit code based interface, output from the commands is not significant.

### Properties

|Property                 |Required                            |Description|
|-------------------------|------------------------------------|-----------|
|command                  |yes                                 |The command to run relative to the watcher manifest directory|
|timeout                  |                                    |How long the command is allowed to run, *10s* default|
|suppress_success_announce|                                    |Do not publish a state JSON document after every run, useful for frequently run items. Still publish on error. Still support regular publish via `announce_interval`|
|environment              |                                    |A list of custom environment variables to set in the form `VAR=val`|

### Behavior

An exec watcher will at *interval* times run the command specified with a few machine specific environment variables set in addition to any set using `environment`. Since version 0.11.1 when the interval is not set or set to 0 the the command will run only on transitions.

|Variable              |Description|
|----------------------|-----------|
|*MACHINE_WATCHER_NAME*|The *name* of the watcher being run|
|*MACHINE_NAME*        |The *name* of the machine being run|
|*PATH*                |Includes the machine directory as last entry|

The command is run with current directory set to the directory where the *machine.yaml* is, when the command exits *0* a *success_transition* fires, when it exits *!0* a *fail_transition* fires. Both cases publish an event announcing the execution.

## Nagios watcher

The *nagios* watcher executes Nagios compatible plugins and emits transitions `OK`, `WARNING`, `CRITICAL` and `UNKNOWN`.  This watcher can be combined with the `exec` watcher to create self healing systems that remediate when in `critical` or `warning` states.

{{% notice tip %}}
This feature is available since *Choria Server 0.15.0*
{{% /notice %}}

### Properties

|Property                 |Required                            |Description|
|-------------------------|------------------------------------|-----------|
|plugin                   |yes                                 |Full path to the Nagios plugin script and it's arguments|
|timeout                  |                                    |How long plugins can run, defaults to 10 seconds. Valid values are of the form 1s, 1m, 1h|

### Behaviour

A `nagios` watcher will at `interval` times run the Nagios check and transition using `OK`, `WARNING`, `CRITICAL` and `UNKNOWN` events.

If the configuration setting `plugin.choria.prometheus_textfile_directory` is set it will write metrics in a format the Prometheus Node Exporter accepts:

```nohighlight
# HELP choria_machine_nagios_watcher_status Choria Nagios Check Status
# TYPE choria_machine_nagios_watcher_status gauge
choria_machine_nagios_watcher_status{name="check_httpd"} 0
choria_machine_nagios_watcher_status{name="check_bacula_db"} 0
# HELP choria_machine_nagios_watcher_last_run_seconds Choria Nagios Check Time
# TYPE choria_machine_nagios_watcher_last_run_seconds gauge
choria_machine_nagios_watcher_last_run_seconds{name="check_httpd"} 1592645406
choria_machine_nagios_watcher_last_run_seconds{name="check_bacula_db"} 1592645405
```

### Remediation

Here's a full working example of a check that includes remediation and the ability to stop checks from happening for a time.  The Puppet defined type `choria::nagios_check` can be used to create this.

```yaml
name: check_httpd
version: 1.0.0
initial_state: unknown

transitions:
  - name: UNKNOWN
    from: [unknown, ok, warning, critical]
    destination: unknown

  - name: OK
    from: [unknown, ok, warning, critical]
    destination: ok

  - name: WARNING
    from: [unknown, ok, warning, critical]
    destination: warning

  - name: CRITICAL
    from: [unknown, ok, warning, critical]
    destination: critical

  - name: MAINTENANCE
    from: [unknown, ok, warning, critical]
    destination: maintenance

  - name: RESUME
    from: [maintenance]
    destination: unknown

watchers:
  - name: check
    type: nagios
    interval: 10s
    state_match: [unknown, ok, warning, critical]
    properties:
      plugin: /usr/lib64/nagios/plugins/check_procs -C httpd -c 1:25

  - name: remediate
    type: exec
    state_match: [critical]
    interval: 10m
    properties:
      command: /sbin/service httpd restart
```

Checks can be stopped using `mco rpc choria_util machine_transition name=check_httpd transition=MAINTENANCE` and resumed using by passing `transition=RESUME` instead.

## Scheduler watcher

The *scheduler* watcher flips between success and fail states based on a set of schedules specified in a crontab like format.  Use it to enter and exit a state on a schedule and combine it with an exec watcher to run commands on a schedule.

### Properties

|Property                 |Required                            |Description|
|-------------------------|------------------------------------|-----------|
|duration                 |yes                                 |How long the scheduler should be in the `success` state once triggered|
|schedules                |yes                                 |A list of crontab like schedules based on which the `success` transitions will fire|

### Behavior

The schedules specified is a list of times when the scheduler will be in success transition, at the end of the trigger time + duration it will fire a fail transition. The fields are like crontab(5), supports ranges, special characters and predefined schedules like `@daily`, see [robfib/cron](https://godoc.org/github.com/robfig/cron) section *CRON Expression Format* for what we'd understand.  We do not support the seconds field.

```yaml
 - name: scheduler
   type: schedule
   fail_transition: stop
   success_transition: start
   state_match: [switched_on, switched_off]
   interval: 1h
   properties:
     duration: 1h
     schedules:
       - "0 8 * * *"
       - "0 12 * * *"
       - "0 17 * * *"
       - "0 20 * * SAT,SUN"
```

The scheduler above will switch on daily at 8am, 12pm and 5pm but also at 8pm on Saturdays and Sundays.  It will stay on for a hour.

If the machine transitions into an eligible *state_match* while a schedule is started it will immediately fire the *success_transition*.  If Choria starts up in the middle of a scheduled period it will be ignored and the next schedule will trigger.  Overlapping schedules is supported.
