+++
title = "Example"
weight = 20
toc = true
+++

An Autonomous Agent consists of a directory with a file *machine.yaml* and an optional set of support scripts and more.

In the concepts section we described a theoretical HVAC management agent, lets look how that might look as a machine.

## Designing the Finite State Machine

Lets try to solve the HVAC problem from our concepts page, to recap we want to have a file lets say *hvac.json* that describes in it thresholds for air quality that you read from sensors in the room. Could be humidity, temperature, co2 levels, whatever you decide. Based on these configured thresholds and the input from your sensors you want to turn the HVAC on and off.

Reasoning about this we determine we have 3 possible states:

 * The *hvac.json* is missing - this we'll call *unknown* state
 * The air quality is within parameters and the HVAC should be off - this we'll call *idle* state
 * The air quality is outside of parameters and the HVAC should be on - this we'll call the *running* state

Lets think about the *transitions*, this is what triggers the HVAC to go off and on and so forth:

 * If at any point the *hvac.json* goes missing we need to move to the *unknown* state, we'll trigger the *variables_unknown* transition
 * If at any point the *hvac.json* changes, we will move to idle state and measure the values, we'll trigger the *variables_changed* transition to achieve this
 * If while measuring the air quality sensors the node falls within good thresholds the *air_good* transition is fired and the HVAC moves to idle
 * If while measuring the air quality the sensors the node falls outside of good thresholds the *air_bad* transition is fired and the HVAC moves to running

![HVAC Machine Transitions](../../hvac_machine_transitions.png)

Lets think about the *watchers*, this is monitors you need to run to inform the system when to fire transitions:

 * When in any state we need to watch the *hvac.json* file, we need to detect the file going missing or the file changing
 * When the machine is *idle* **or** *running* we need to monitor the sensors and trigger either *air_good* or *air_bad*
 * When the machine is in *idle* **or** *unknown* state we should keep the HVAC turned off
 * When the machine is in *running* state we should keep the HVAC turned on


## Disk layout

Machines are stored in directories, the example below would be stored like this:

```nohighlight
hvac
├── machine.yaml
├── hvac.json
├── monitor.sh
├── off.sh
└── on.sh
```

When you use scripts in *exec* watchers or reference files in *file* watchers the paths are relative to your *machine.yaml*.  At present the only requirement is that a *machine.yaml* exist in the directory the rest are optional.

## Manifest

The manifest here describe the machine we designed above, the *splay_start* item is optional, everything else is required.

{{% notice tip %}}
A [JSON schema](https://choria.io/schemas/choria/machine/v1/manifest.json) describes these files and you can configure some editors to validate the YAML file based on that. The command `choria machine validate` can validate a *machine.yaml* against this schema.
{{% /notice %}}

The scripts called are not shown here, they simply have to exit 0 on success or 1 on failure, everything else is ignored.

```yaml
name: HVAC

# Must be SemVer format
version: "1.0.0"

# The state the machine starts in
initial_state: unknown

# Will wait a random period up to this many seconds before starting, defaults to 0
splay_start: 30

# Creates all the valid transitions this machine can receive, we do
# not specifically need to list all the states as they are inferred
# from the "from" and "destination" lines
transitions:
  - name: variables_unknown
    from: [unknown, idle, running]
    destination: unknown

  - name: variables_changed
    from: [unknown, idle, running]
    destination: idle

  - name: air_good
    from: [idle, running]
    destination: idle

  - name: air_bad
    from: [idle, running]
    destination: running

watchers:
  # checks the hvac.json exist and if its changed, on change
  # success_transition is fired, if the file is missing
  # fail_transition is fired.
  - name: variables
    type: file
    state_match: [unknown, idle, running]
    fail_transition: variables_unknown
    success_transition: variables_changed
    interval: "5s"
    properties:
        path: hvac.json

  # turns the HVAC off, no special handling for failures - it would
  # stay in the state it was and keep retrying - on success it moves
  # the machine to idle
  - name: off
    type: exec
    state_match: [unknown, idle]
    success_transition: idle
    interval: "5m"
    properties:
        command: off.sh

  # turns the HVAC on, this is only valid in the running state and
  # it runs regularly to keep the HVAC on in case someone fiddles
  # with the switch
  - name: on
    type: exec
    state_match: [running]
    interval: "5m"
    properties:
        command: on.sh

  # monitors the air, exits with 0 if the air is good and 1 if its bad
  - name: monitor
    type: exec
    state_match: [idle, running]
    success_transition: air_good
    fail_transition: air_bad
    interval: "1m"
    properties:
        command: monitor.sh
```

## Validating

You can validate the *machine.yaml* by running *choria machine validate hvac* where *hvac* is the directory with your machine.

## Graph

The graph below can be generated using the *choria machine graph hvac/* command based on the previous layout.

<img src="../../hvac_fsm.svg" />

You can use this to verify the structure of your machine and what transitions are supported etc, it's in the Graphviz dot format and can be viewed using many online or offline viewers.

## Running locally

You can run the machine locally on your desktop during testing using the *choria machine run hvac* command, where *hvac* is a directory that matches the above layout.

## Running inside Choria Server

Machines are hosted within your Choria Server and are read from disk. There isn't much to configure:

```ini
plugin.choria.machine.store = /etc/choria/machines
```

On start every machine found in directories there will be started and hosted forever, they will log and publish events to the middleware.  Machines with *splay_start* set as in the example above will sleep randomly up to this many seconds before starting, if you are running for example a cron style job on a large estate this can assist with spreading the load.

Updates to this directory will be detected, machines stopped and reloaded from disk on demand without restarting the Choria Server.