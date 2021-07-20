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
|fail_transition   |no |If set this event fires on failure|
|success_transition|no |If set this event fires on success|
|interval          |no |Runs the watcher every interval, valid intervals are of the form *1s*, *1m*, *1h*|
|announce_interval |no |Announce the current state of the watcher regularly, valid intervals are of the form *1s*, *1m*, *1h*|
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

|Property                 |Required|Description|
|-------------------------|--------|-----------|
|command                  |yes     |The command to run relative to the watcher manifest directory|
|timeout                  |        |How long the command is allowed to run, *10s* default|
|suppress_success_announce|        |Do not publish a state JSON document after every run, useful for frequently run items. Still publish on error. Still support regular publish via `announce_interval`|
|environment              |        |A list of custom environment variables to set in the form `VAR=val`|
|governor                 |        |Use the named [Choria Concurrency Governor](../../streams/governor) to control how many concurrent actions can be taken|
|governor_timeout         |        |How long to attempt to gain a slot on the governor, defaults to *5m*|
|parse_as_data            |        |Attempt to parse the data as JSON data and store each JSON key in a matching data item|                   

### Data

Choria Autonomous Agents can store data on a per machine basis. The data be written using the *kv* store watcher or the output
from an exec command can be parsed as JSON and stored in the machine.

Generally it's best to think of data keys and values as strings, but we do support any JSON data type for data coming from an
*exec* watcher.

Any *exec* that runs has access to all the machine data. While we persist the data to disk between runs and restarts it's best
to think of the data as ephemeral. It's wont be there on start and we will delete corrupt data.  Your Autonomous Agent should 
be able to run without data - by gathering it or creating it on demand.

The `environment`, `command` and `governor` properties support looking up facts and data:

```yaml
properties:
  governor: APP_{{ lookup "facts.location" "DEFAULT" }}
```

Will look up the fact `location` within the Choria Server facts using `DEFAULT` when the fact is not set.  You can also access the `data` structure to get machine data. Nested data can be looked up using `gjson` syntax, for example `{{ lookup "facts.os.distro.codename" "UNKNOWN }}`.

Additional to the `lookup` function we also have `Title` (Title Case A String), `Capitalize` (Same as Title), `ToLower`, `ToUpper`, `StringJoin` (comma joins a list of strings), `Base64Encode` and `Base64Decode`.

These templates are expanded every time the data is needed so if you reference machine data you can make it dynamic - every time a command,
governor etc is needed the template is newly expanded.

### Behavior

An exec watcher will at *interval* times run the command specified with a few machine specific environment variables set in addition to any set using `environment`. Since version 0.11.1 when the interval is not set or set to 0 the the command will run only on transitions.

|Variable              |Description|
|----------------------|-----------|
|*MACHINE_WATCHER_NAME*|The *name* of the watcher being run|
|*MACHINE_NAME*        |The *name* of the machine being run|
|*PATH*                |Includes the machine directory as last entry|
|*WATCHER_DATA*        |A path to a temporary file that holds a copy of machine data in JSON format|
|*WATCHER_FACTS*       |A path to a temporary file that holds a copy of all known facts about a machine in JSON format|

The command is run with current directory set to the directory where the *machine.yaml* is, when the command exits *0* a *success_transition* fires, when it exits *!0* a *fail_transition* fires. Both cases publish an event announcing the execution.

If a *governor* is configured the watcher will try to obtain a slot in the Governor before executing the command, it will timeout after *governor_timeout* has passed.

To create a [Choria Concurrency Governor](../../streams/governor) must be enabled on the broker and the named Governor should have been created using `choria governor add`.

## Key-Value store 

The *kv* watcher watches a Choria Key-Value Store key and act on changes, updating the Machine data store with values and optionally
performing transitions on change.

{{% notice tip %}}
This feature is available since *Choria Server 0.23.0*
{{% /notice %}}

Polling the Key-Value store is less resource intensive than watching, polling can be slower though.  In general this feature
should be used on thousands of machines maximum rather than 10s of thousands.

### Properties

|Property|Required|Description|
|--------|--------|-----------|
|bucket  |yes     |The name of the bucket to watch|
|key     |        |Watch a specific key in the bucket|
|mode    |        |Either *poll* or *watch*|
|bucket_prefix|   |Store the data in the machine data store with a prefix matching the bucket name, on by default|

### Behavior

The *kv* watcher will at *interval* fetch the value of a key in *poll* mode or actively watch buckets for change. Any change
on the watched key will be stored in the Machine data and made available to other watchers like the *exec* one.

If *bucket_prefix* is true (the default), the data will be stored like *BUCKET_KEY* in the data, otherwise just *KEY*.

The *fail_transition* is called for any Key-Value retrieval failure, *success_transition* on any data change - including
if a watched key is deleted.

It does not announce state regularly or on state changes.

## Nagios watcher

The *nagios* watcher executes Nagios compatible plugins and emits transitions `OK`, `WARNING`, `CRITICAL` and `UNKNOWN`.  This watcher can be combined with the `exec` watcher to create self healing systems that remediate when in `critical` or `warning` states.

### Properties

|Property                 |Required  |Description|
|-------------------------|----------|-----------|
|builtin                  |no        |Builtin check plugin, `heartbeat`, `choria_status` or `goss` are supported now|
|plugin                   |no        |Full path to the Nagios plugin script and it's arguments|
|timeout                  |          |How long plugins can run, defaults to 10 seconds. Valid values are of the form 1s, 1m, 1h|
|annotations              |no        |Additional annotations to apply to a specific check as a `map[string]string` JSON object|
|gossfile                 |no        |A check specific goss file, else the system wide one is used|
|last_message             |no        |For the `choria_status` builtin check, how long ago the last RPC message should have been received|

When setting the plugin one can load override data from a JSON file defined in `plugin.scout.overrides`:

```yaml
name: check_pki
watchers:
- name: check
  properties:
    plugin: '/check -W {{ o "warn" 10 }} -C {{ o "crit" 20 }}'
```

This will load the data `{"check_pki": {"warn": 15}}` from the `plugin.scout.overrides` path setting `warn` to 15 and `crit` would be 20 as there is no override.

Either `plugin` or `builtin` has to be set, builtin can only be `heartbeat` or `goss` at the moment.

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

### Goss

The nagios watcher can run [goss](https://github.com/aelsabbahy/goss) checks on a regular basis and publish events based on the outcome.

Goss is a system allowing you to describe your system state to quite a lot of detail and it will then validate the system against that.

An example `gossfile` can be seen here and is typically saved as `/etc/choria/goss.yaml` - a path configured using the `plugin.scout.goss.file` setting.

```yaml
port:
  tcp:22:
    listening: true
    ip:
    - 0.0.0.0
  tcp6:22:
    listening: true
    ip:
    - '::'
service:
  sshd:
    enabled: true
    running: true
user:
  sshd:
    exists: true
    uid: 74
    gid: 74
    groups:
    - sshd
    home: /var/empty/sshd
    shell: /sbin/nologin
group:
  sshd:
    exists: true
    gid: 74
process:
  sshd:
    running: true
```

With this content in the file, a `goss` builtin check will validate all these properties regularly.

### Remediation

Here's a full working example of a check that includes remediation and the ability to stop checks from happening for a time.  The Puppet defined type `choria::scout_check` can be used to create this.

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

## Home Kit watcher

The `homekit` watcher creates an Apple Home Kit Button resource that can be activated from iOS devices and Siri. When the button is pressed on a `success_transition` is fired and when off a `fail_transition`.

This watcher requires write access to a `homekit` directory within the machine directory to store state.

### Properties

|Property                 |Required |Description|
|-------------------------|---------|-----------|
|pin                      |yes      |The pin to enter when adding the button to Home App|
|serial_number            |         |The serial number to report to Home Kit|
|model                    |         |The model to report to Home Kit, defaults to `Autonomous Agent`|
|setup_id                 |         |A Home Kit set up id to report|
|initial                  |         |The initial state of the button, either `on` or `off`|
|on_when                  |         |When the machine is in any of these states the button will be reported as `on` to Home Kit|
|off_when                 |         |When the machine is in any of these states the button will be reported as `off` to Home Kit|
|disable_when             |         |When the machine is in any of these states the Home Kit integration will shut down and the button will be unreachable|

### Behavior

This creates an Apple Home Kit button reported as `Extractor`, it will be a simple on/off style button that fires `success_transition` when pressed on and `fail_transition` when pressed off.

When another watcher, or external RPC event, transitions the machine to different states this button can flip to on or off dependant on the states listed in `on_when` and `off_when`.

```yaml
- name: extractor
  type: homekit
  state_match:
    - unknown
    - "on"
    - "off"
    - 2hours_on
    - 2hours_off
  success_transition: override_on
  fail_transition: override_off
  properties:
    pin: "12345679"
    on_when: [2hours_on, "on"]
    off_when: [2hours_off, unknown, "off"]
    disable_when: [pause]
```

## Timer watcher

The `timer` watcher starts a timer when the state machine transitions into a state that it's matched on and emits a transition event when the timer end.

This can be used to create systems like a maintenance window that automatically expire.

### Properties

|Property                 |Required                            |Description|
|-------------------------|------------------------------------|-----------|
|timer                    |yes                                 |How long the timer should run for, triggers `fail_transition` at the end of the timer|

### Behavior

The timer will start whenever the machine enters a state listed in `state_match`, once the timer reach the end it will trigger `fail_transition` if set. If, while active, the machine transitions from one state in `state_match` to another also in `state_match` the timer will reset.

```yaml
- name: 2hours_off_timer
  type: timer
  success_transition: override_ends
  state_match:
  - 2hours_off
  properties:
    timer: 2h 
```

## Scheduler watcher

The *scheduler* watcher flips between success and fail states based on a set of schedules specified in a crontab like format.  Use it to enter and exit a state on a schedule and combine it with an exec watcher to run commands on a schedule.

### Properties

|Property                 |Required                            |Description|
|-------------------------|------------------------------------|-----------|
|start_splay              |no                                  |Sleep a random period before initiating the schedule, expressed as a duration like `1m`. Should be no more than half the `duration`|
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
     start_splay: 1m
     schedules:
       - "0 8 * * *"
       - "0 12 * * *"
       - "0 17 * * *"
       - "0 20 * * SAT,SUN"
```

The scheduler above will switch on daily at 8am, 12pm and 5pm but also at 8pm on Saturdays and Sundays.  It will stay on for an hour. Before starting it will sleep a random period between 0 and 1 minute. 

If the machine transitions into an eligible *state_match* while a schedule is started it will immediately fire the *success_transition*.  If Choria starts up in the middle of a scheduled period it will be ignored and the next schedule will trigger.  Overlapping schedules is supported.

## Metric watcher

The *metric* watcher periodically run a command and publish metrics found in its output to Prometheus. Both a Choria specific metric format and Nagios Perfdata is supported.

### Properties

|Property                 |Required                            |Description|
|-------------------------|------------------------------------|-----------|
|command                  |yes                                 |Path to the command to run to retrieve the metric|
|interval                 |yes                                 |Go duration for how frequently to gather metrics|
|labels                   |no                                  |key=value pairs of strings of additional labels to add to gathered metrics|

### Behaviour

The plugin supports 2 data formats, one Choria specific one and the commonly used Nagios Perfdata format. 

If you're writing your own gathering scripts we suggest the Choria format.

The watcher will at `interval` run the `command` and create Prometheus data. The labels from the specific output is augmented by `labels`, the `labels` given here will override those from the `command`.

### Choria Metric Format

Given output as seen here the following metrics will be produced - we added an additional label `home` in the watcher properties, the watcher is called `kasa`:

```json
{
  "labels": {
    "alias": "Geyser",
    "id": "xxx",
    "model": "HS110(UK)"
  },
  "metrics": {
    "current_amp": 0.020999999716877937,
    "on_seconds": 0,
    "power_state": 0,
    "power_watt": 0,
    "total_watt": 0.3330000042915344,
    "voltage_volt": 239.38099670410156
  }
}
```

```nohighlight
# HELP choria_machine_metric_watcher_kasa_power_watt Choria Metric
# TYPE choria_machine_metric_watcher_kasa_power_watt gauge
choria_machine_metric_watcher_kasa_power_watt{alias="geyser",id="xxx",model="hs110(uk)",location="home"} 0.000000
# HELP choria_machine_metric_watcher_kasa_total_watt Choria Metric
# TYPE choria_machine_metric_watcher_kasa_total_watt gauge
choria_machine_metric_watcher_kasa_total_watt{alias="geyser",id="xxx",model="hs110(uk)",location="home"} 0.333000
# HELP choria_machine_metric_watcher_kasa_voltage_volt Choria Metric
# TYPE choria_machine_metric_watcher_kasa_voltage_volt gauge
choria_machine_metric_watcher_kasa_voltage_volt{alias="geyser",id="xxx",model="hs110(uk)",location="home"} 237.123001
# HELP choria_machine_metric_watcher_kasa_choria_runtime_seconds Choria Metric
# TYPE choria_machine_metric_watcher_kasa_choria_runtime_seconds gauge
choria_machine_metric_watcher_kasa_choria_runtime_seconds{id="xxx",model="hs110(uk)",location="home",alias="geyser"} 0.148666
# HELP choria_machine_metric_watcher_kasa_current_amp Choria Metric
# TYPE choria_machine_metric_watcher_kasa_current_amp gauge
choria_machine_metric_watcher_kasa_current_amp{alias="geyser",id="xxx",model="hs110(uk)",location="home"} 0.021000
# HELP choria_machine_metric_watcher_kasa_on_seconds Choria Metric
# TYPE choria_machine_metric_watcher_kasa_on_seconds gauge
choria_machine_metric_watcher_kasa_on_seconds{alias="geyser",id="xxx",model="hs110(uk)",location="home"} 0.000000
# HELP choria_machine_metric_watcher_kasa_power_state Choria Metric
# TYPE choria_machine_metric_watcher_kasa_power_state gauge
choria_machine_metric_watcher_kasa_power_state{location="home",alias="geyser",id="xxx",model="hs110(uk)"} 0.000000
```

### Nagios Metric Format

The Nagios format takes standard Perfdata format as seen here:

```nohighlight
OK: last run 24 minutes ago with 0 failed resources 0 failed events and currently enabled|time_since_last_run=1491s;1;1;0 failed_resources=0;;;0 failed_events=0;;;0 last_run_duration=59.67;;;0
```

This will produce output as above for metrics `choria_machine_metric_watcher_puppet_time_since_last_run` and so forth.
