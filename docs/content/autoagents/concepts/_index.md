+++
title = "Concepts"
weight = 10
toc = true
+++

Autonomous Agents (also Choria Machines) are implemented as [Finite State Machines](https://en.wikipedia.org/wiki/Finite-state_machine) that you describe in a file *machine.yaml*.

## Terminology

### State

A state is a named stable state the machine can find itself in. If you are managing the room air quality using an HVAC system you would have states *unknown*, *idle* and *running*.

### Transitions

The machine can receive events, for example in the HVAC example you would periodically check the air quality and should it be considered bad using your own criteria you'd fire the *air_bad* event that would move the machine into the *running* state. Likewise when the air is good you'd fire the *air_good* event to move to the *idle* state.

Finally it would have a *variables_changed* event that can fire if the JSON file with your air quality requirements in it change, and a *variables_unknown* event should the JSON file be missing.

These events - *air_good*, *air_bad*, *variables_unknown* and *variables_changed* - are called transitions.

### Watchers

In the HVAC example above you would have a 4 watchers.

First you'd have a *file* watcher to check the JSON file containing your measurements. If the file is missing it would trigger a *variables_unknown* transition keeping the macine in the *unknown* state. If the file is there or changes it would fire a *variables_changed* transition moving the machine to *idle* state.

Second you'd have a script that can turn the HVAC off and keep it off, this watcher would be valid only in the *unknown* and *idle* states and would run regularly. This ensures the HVAC is off when unknown and keeps it off should someone fiddle the switches.

Third you'd have a script that turn the HVAC on and keep it on, this would only be executable in the *running* state and would be executing regularly there to ensure it stays on.

Finally you'd write a small script that would query your sensors and determine if the air is good or bad. This is an *exec* watcher that is valid in any of the *idle* or *running* states and it would run on a regular interval like for example *1m*.

### Events

Choria has a standard set of events - alive, startup, shutdown, provisioned and more - state transitions and state announcement will publish their own event types. Dashboards will be made that can passively observe a running network of machines.

### State Machine

A State Machine combines *states* and *transitions* to describe every possible situation the managed item might find itself in and how it gets to that situation or state.

