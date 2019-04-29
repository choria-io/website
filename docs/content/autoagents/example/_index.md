+++
title = "Example"
weight = 20
toc = true
+++

An Autonomous Agent consists of a directory with a file *machine.yaml* and an optional set of support scripts and more.

In the concepts section we described a theoretical HVAC management agent, lets look how that might look as a machine.

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

## Graph

The graph below can be generated using the *choria machine graph hvac/* command based on the previous layout.

<img src="/docs/hvac_fsm.svg" />

## Manifest

The manifest here describe the machine that was first presented in the concepts page, it manages a simple HVAC system based on air quality. All keys in the document here are required.

{{% notice tip %}}
A [JSON schema](https://choria.io/schemas/choria/machine/v1/manifest.json) describes these files and you can configure some editors to validate the YAML file based on that. The command `choria machine validate` can validate a *machine.yaml* against this schema.
{{% /notice %}}

The scripts called are not shown here, they simply have to exit 0 on success or 1 on failure, everything else is ignored.

```yaml
name: HVAC
version: "1.0.0"
initial_state: unknown

# Creates all the valid events this machine can receive, we do
# not specifically need to list all the states that is inferred
# from the "from" and "destination" lines.
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

## Running locally

You can run the machine locally on your desktop during testing using the *choria machine run hvac* command, where *hvac* is a directory that matches the above layout.

## Running inside Choria Server

This is not yet supported