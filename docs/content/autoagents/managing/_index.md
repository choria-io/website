+++
title = "Managing Instances"
weight = 30
toc = true
+++

Running instances of Choria Autonomous Agents can be observed using the events they publish in JSON - and we provide a utility to assist with that - additionally they can be managed via the normal Choria RPC interface, this feature will be expanded a lot in the future right now it's quite limited.

### Watching the real time events and transitions

The various watchers and machines publish events like on every State Transition, Watcher execution or regular Watcher state announcements based on the specification.  These events can be viewed using the `choria machine watch` command:

#### All Events

```nohighlight
$ choria machine watch
INFO[0000] Viewing transitions on topic choria.machine.transition
INFO[0000] Viewing watcher states on topic choria.machine.watcher.*.state
example.net TestMachine transitioned via event fire_1: unknown => one
example.net TestMachine#true_1 command: /usr/bin/true, previous: success ran: 0.001s
example.net TestMachine transitioned via event fire_2: one => two
example.net TestMachine#true_2 command: /usr/bin/true, previous: success ran: 0.001s
example.net TestMachine#true_2 command: /usr/bin/true, previous: success ran: 0.001s
example.net DockerExample#manifest path: /etc/choria/machine/docker/manifest.json, previous: changed
example.net DockerExample transitioned via event absent: unknown => deploy
example.net DockerExample transitioned via event deployed: deploy => monitor
example.net DockerExample#deploy command: ./deploy.rb, previous: success ran: 0.085s
example.net DockerExample#check_running command: ./check_running.rb, previous: success ran: 0.085s
example.net DockerExample#monitor command: ./monitor.rb, previous: success ran: 1.352s
```

#### Only Transitions

```nohighlight
$ choria machine watch --transitions
INFO[0000] Viewing transitions on topic choria.machine.transition
example.net TestMachine transitioned via event fire_1: unknown => one
example.net TestMachine transitioned via event fire_2: one => two
example.net DockerExample transitioned via event absent: unknown => deploy
example.net DockerExample transitioned via event deployed: deploy => monitor
```

#### Only Watcher States

```nohighlight
$ choria machine watch --watchers
INFO[0000] Viewing watcher states on topic choria.machine.watcher.*.state
example.net TestMachine#true_1 command: /usr/bin/true, previous: success ran: 0.002s
example.net TestMachine#true_2 command: /usr/bin/true, previous: success ran: 0.001s
example.net TestMachine#true_2 command: /usr/bin/true, previous: success ran: 0.002s
example.net DockerExample#manifest path: /etc/choria/machine/docker/manifest.json, previous: changed
example.net DockerExample#deploy command: ./deploy.rb, previous: success ran: 0.074s
example.net DockerExample#check_running command: ./check_running.rb, previous: success ran: 0.059s
example.net DockerExample#monitor command: ./monitor.rb, previous: success ran: 1.321s
```

You can filter these further by one or more `--type` flags.

#### Raw JSON

These events are published on the middleware as JSON, you can view these raw:

{{% notice tip %}}
JSON Schemas for all published messages exist, for example `io.choria.machine.v1.transition` would be `https://choria.io/schemas/choria/machine/v1/transition.json`, the messages are in the data field of [CloudEvents](https://cloudevents.io) messages.
{{% /notice %}}

```nohighlight
$ choria tool sub "choria.machine.>"
Waiting for messages from topic choria.machine.> on nats://example:4222

---- 16:46:55 on topic choria.machine.watcher.exec.state
{"protocol":"io.choria.machine.watcher.exec.v1.state","identity":"example.net","id":"79933942-6f5e-4cff-95e9-24ee1604bd9b","version":"0.0.1","timestamp":1557326815541428330,"type":"exec","machine":"DockerExample","name":"check_running","command":"./check_running.rb","previous_outcome":"success","previous_run_time":62199712}
```

State transitions are published on `choria.machine.transition` topic while watcher states are published on, for example, `choria.machine.watcher.file.state` where you replace *file* with the watcher type. Above example using `choria.machine.>` watches all events.

### Request current state

Each node hosting Machines will return their list of instances and details about each, significantly you note the current state and what are possible transition.

If you are coding management tools againt this use the `id`, `name`, `version`, `path` and any combination of these to later interact with a specific machine or be sure to add enough identifying information since it's possible to run the same machine with the same name at different versions in the same Choria Server.

To assist in managing a running Machine you can also see here what are available transitions in its current state.

```nohighlight
$ mco rpc choria_util machine_states
Discovering hosts using the choria method .... 1

 * [ ============================================================> ] 1 / 1


example.net
      Machine IDs: ["143c3487-396d-4134-adba-ff0f0058e824",
                    "e873a2ef-b55d-494f-a816-a9eb4ed9d7b8"]
    Machine Names: ["DockerExample 0.0.1", "TestMachine 0.0.1"]
   Machine States: {"143c3487-396d-4134-adba-ff0f0058e824"=>
                     {"name"=>"DockerExample",
                      "version"=>"0.0.1",
                      "state"=>"monitor",
                      "path"=>"/etc/choria/machine/docker",
                      "id"=>"143c3487-396d-4134-adba-ff0f0058e824",
                      "start_time"=>1556901166,
                      "available_transitions"=> [
                        "unhealthy",
                        "maintenance",
                        "absent",
                        "no_manifest"
                      ]},
                    "e873a2ef-b55d-494f-a816-a9eb4ed9d7b8"=>
                     {"name"=>"TestMachine",
                      "version"=>"0.0.1",
                      "state"=>"two",
                      "path"=>"/etc/choria/machine/test",
                      "id"=>"e873a2ef-b55d-494f-a816-a9eb4ed9d7b8",
                      "start_time"=>1556901167,
                      "available_transitions"=>[]}}


Summary of Machine Names:

     TestMachine 0.0.1 = 1
   DockerExample 0.0.1 = 1


Finished processing 1 / 1 hosts in 178.88 ms
```

### Requesting a state change

You can force initiate a `Transition` by name on a specific machine, this is useful if your machine has a state where it effectively enters a maintenance mode for instance when you do not wish to have it remediate down components while you do maintenance.

These transition requests can of course fail - your machine might be in a state where the transition you are requesting is not valid, in that case the RPC request will fail with appropriate error state.

```nohighlight
$ mco rpc choria_util machine_transition name=DockerExample transition=resume
Discovering hosts using the choria method .... 1

 * [ ============================================================> ] 1 / 1

Finished processing 1 / 1 hosts in 144.41 ms
```

In addition to `name` you can also pass `version`, `path` or even the instance ID via `instance`.  These criteria are searched in an `AND` manner in case you run multiple instances of the same machine.
