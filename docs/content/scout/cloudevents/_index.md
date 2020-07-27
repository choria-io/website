+++
title = "CloudEvents"
weight = 30
+++

Scout publishes a lot of data as [CloudEvents](https://cloudevents.io/) format to the NATS middleware. This data can 
be gathered in real time or stored in our - currently preview - Streaming server for analysis and integration purposes.

## Observing

Choria includes tools already for viewing events produced by the Autonomous Agent and we are adding some Scout specific
tools that focus on this particular use case.

### Status Events

We have an initial Scout watch tool that can observe the real time event stream and also retrieve history from our
Technology Preview streaming technology.

Here invoked with `choria scout watch --identity dev1.devco.net`

![Scout Watch](../../scout-watch.png)

Previously we added generic Autonomous Agent event viewers, those too can show Scout events:

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
    "identity": "dev1.example.net",
    "id": "3901412f-7453-440f-bee0-19301e059a6d",
    "version": "1.0.0",
    "timestamp": 1595879087,
    "type": "nagios",
    "machine": "check_mailq",
    "name": "check",
    "plugin": "/usr/lib64/nagios/plugins/check_mailq -M exim -w 5 -c 10",
    "status": "OK",
    "status_code": 0,
    "output": "OK: mailq (0) is below threshold (5/10)|unsent=0;5;10;0",
    "check_time": 1595879087,
    "perfdata": [
      {
        "unit": "",
        "label": "unsent",
        "value": 0
      }
    ],
    "runtime": 0.041653168,
    "history": [
      {
        "execute": "2020-07-27T18:34:47.059023977Z",
        "status": 0,
        "perfdata": [
          {
            "unit": "",
            "label": "unsent",
            "value": 0
          }
        ]
      },
      {
        "execute": "2020-07-27T18:39:47.058776768Z",
        "status": 0,
        "perfdata": [
          {
            "unit": "",
            "label": "unsent",
            "value": 0
          }
        ]
      },
      {
        "execute": "2020-07-27T18:44:47.058770221Z",
        "status": 0,
        "perfdata": [
          {
            "unit": "",
            "label": "unsent",
            "value": 0
          }
        ]
      },
      {
        "execute": "2020-07-27T18:49:47.058799245Z",
        "status": 0,
        "perfdata": [
          {
            "unit": "",
            "label": "unsent",
            "value": 0
          }
        ]
      }
    ]
  },
  "id": "ca330f1c-4d75-4688-86d4-6e6775672f79",
  "source": "io.choria.machine",
  "specversion": "1.0",
  "subject": "dev1.example.net",
  "time": "2020-07-27T19:44:47Z",
  "type": "io.choria.machine.watcher.nagios.v1.state"
}
```

The document in the `data` field complies to the [io.choria.machine.watcher.nagios.v1.state](https://choria.io/schemas/choria/machine/watcher/nagios/v1/state_notification.json)
schema.

