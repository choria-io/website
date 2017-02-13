+++
title = "Statistics"
toc = true
weight = 225
+++

MCollective supports various registration plugins, these are traditionaly used to build
caches of information about the collective but this is now replaced with PuppetDB.

Choria includes a Registration plugin that writes statistics about the daemon to a JSON
file, by default in the directory the *mcollective.log* is written in a file called
*choria-stats.json*.

{{% notice tip %}}
This is included since *0.0.22*
{{% /notice %}}

### Configuring

You have to configure MCollective to use this plugin:

```yaml
mcollective::server_config:
  registerinterval: "10"
  registration: "Choria"
```

And optionally you can configure the plugin to write to a specific file:

```yaml
mcollective_choria::server_config:
  "registration.file": "/path/to/choria-stats.json"
```

### Example

The statistics kept looks like this:

```json
{
    "timestamp": 1486842703,
    "identity": "dev1.example.net",
    "version": "2.9.1",
    "stats": {
        "stats": {
            "validated": 5,
            "unvalidated": 0,
            "passed": 5,
            "filtered": 0,
            "starttime": 1486832930,
            "total": 5,
            "ttlexpired": 0,
            "replies": 5
        },
        "threads": [
            "#<Thread:0x0000000081a628 sleep>",
            "#<Thread:0x0000000163e218 sleep>",
            "#<Thread:0x007f5e61742298 sleep>",
            "#<Thread:0x007f5e61742158 sleep>",
            "#<Thread:0x007f5e61742068 sleep>",
            "#<Thread:0x007f5e604a2958 run>"
        ],
        "pid": 14289,
        "times": {
            "utime": 8.58,
            "stime": 3.61,
            "cutime": 0.0,
            "cstime": 0.0
        },
        "agents": [
            "puppet",
            "service",
            "package",
            "filemgr",
            "discovery",
            "rpcutil"
        ]
    },
    "nats": {
        "connected_server": "nats://nats1.example.net:4222",
        "stats": {
            "in_msgs": 5,
            "out_msgs": 5,
            "in_bytes": 23120,
            "out_bytes": 4520,
            "reconnects": 0
        }
    }
}
```
