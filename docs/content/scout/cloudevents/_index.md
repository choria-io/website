+++
title = "CloudEvents"
weight = 30
+++

Scout publishes a lot of data as [CloudEvents](https://cloudevents.io/) format to the NATS middleware. This data can 
be gathered in real time or stored in our - currently preview - Streaming server for analysis and integration purposes.

## Observing

We will soon add CLI utilities to view the events in a friendly manner, for now our generic event integrations can be
used to see the events.

### Status Events

These are events published for every single check that gets done:

```
$ choria machine watch
dev1.devco.net heartbeat#check OK: 1594915959
dev1.devco.net check_mailq#check OK: OK: mailq (0) is below threshold (5/10)|unsent=0;5;10;0
```

You can limit the view to just transitions - ie. when a check moves from `OK` to `CRITICAL` or other states - by passing
the `--transitions` option.

These events are published to the middleware on the subjects `choria.machine.transition` and `choria.machine.watcher.nagios.state`
in JSON CloudEvent format.

```yaml
$ choria tool sub choria.machine.watcher.nagios.state
---- 18:15:55
{
  "data": {
    "protocol": "io.choria.machine.watcher.nagios.v1.state",
    "identity": "dev1.devco.net",
    "id": "57e1a6c3-b4e8-4cb6-a893-d9c7ccb0d595",
    "version": "1.0.0",
    "timestamp": 1594916155,
    "type": "nagios",
    "machine": "check_mailq",
    "name": "check",
    "plugin": "/usr/lib64/nagios/plugins/check_mailq -M exim -w {{ o \"warn\" 5 }} -c {{ o \"crit\" 10 }}",
    "status": "OK",
    "status_code": 0,
    "output": "OK: mailq (0) is below threshold (5/10)|unsent=0;5;10;0",
    "check_time": 1594916155,
    "perfdata": [
      {
        "unit": "",
        "label": "unsent",
        "value": 0
      }
    ],
    "runtime": 0.036097902,
    "history": [
      0,
      0,
      0,
      0,
      0,
      0,
      0,
      0,
      0,
      0,
      0,
      0,
      0,
      0,
      0,
      0
    ]
  },
  "id": "9613d3d3-927a-4659-9b24-c1578d93212c",
  "source": "io.choria.machine",
  "specversion": "1.0",
  "subject": "dev1.devco.net",
  "time": "2020-07-16T16:15:55Z",
  "type": "io.choria.machine.watcher.nagios.v1.state"
}
```

The document in the `data` field complies to the [io.choria.machine.watcher.nagios.v1.state](https://choria.io/schemas/choria/machine/watcher/nagios/v1/state_notification.json)
schema.

