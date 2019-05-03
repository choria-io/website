+++
title = "Managing Instances"
weight = 30
toc = true
+++

Running instances of Choria Autonomous Agents can be managed via the normal Choria RPC interface, this feature will be expanded a lot in the future right now its quite limited.

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
````

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

### Request current state

Each node hosting Machines will return their list of instances and details about each, significantly you note the current state and what are possible transition.

If you are coding management tools againt this use the `id` to later interact with a specific machine or be sure to add enough identifying information since it's possible to run the same machine with the same name at different versions in the same Choria Server.

```nohighlight
$ mco rpc choria_util machine_states
Discovering hosts using the choria method .... 1

 * [ ============================================================> ] 1 / 1


example.net
      Machine IDs: ["0fc2c13a-d628-43e4-96b3-9c47e3f5e19e"]
   Machine States: {"0fc2c13a-d628-43e4-96b3-9c47e3f5e19e"=>
                     {"name"=>"DockerExample",
                      "version"=>"0.0.1",
                      "state"=>"monitor",
                      "path"=>"/etc/choria/machine/docker",
                      "id"=>"0fc2c13a-d628-43e4-96b3-9c47e3f5e19e",
                      "start_time"=>1556832746,
                      "available_transitions"=>[
                        "unhealthy",
                        "no_manifest",
                        "maintenance",
                        "absent"
                      ]}}

Finished processing 1 / 1 hosts in 121.22 ms
```

### Requesting a state change

You can force initiate a `Transition` by name on a specific machine, this is useful if your machine have a state where it effectively enters a maintenance mode for instance when you do not wish to have it remediate down components while you do maintenance.

These transition requests can of course fail - your machine might be in a state where the transition you are requesting is not valid, in that case the RPC request will fail with appropriate error state.

```nohighlight
$ mco rpc choria_util machine_transition name=DockerExample transition=resume
Discovering hosts using the choria method .... 1

 * [ ============================================================> ] 1 / 1

Finished processing 1 / 1 hosts in 144.41 ms
```

In addition to `name` you can also pass `version`, `path` or even the instance ID via `instance`.  These criteria are search in an `AND` manner in case you run multiple instances of the same machine.