+++
title = "Node Management"
weight = 40
+++

Each Choria managed node expose a RPC API accessible over the Choria network for managing the checks on the node.

Avaialble tasks include:

 * _checks_ - listing checks
 * _trigger_ - trigger an instant check
 * _maintenance_ - pause regular checks for a specific check
 * _resume_ - resume previously paused regular checks
 * _goss_validate_ - performs a [goss](https://github.com/aelsabbahy/goss) validation on a node
 
In time we will include a CLI for interacting with these APIs, today we publish a Golang API and it's usable from the 
CLI.

## CLI

On the CLI the API can be accessed using the normal _choria req_ command:

### Listing checks

This is a list of running checks and their status:

```nohighlight
$ choria req scout checks -I dev1.example.net
Discovering nodes .... 1

1 / 1    0s [====================================================================] 100%

dev1.example.net
   Checks: [
              {
                 "name": "mailq",
                 "start_time": 1594911784,
                 "state": "OK",
                 "version": "1.0.0"
              }
           ]

Finished processing 1 / 1 hosts in 1s
```

### Triggering checks

One or all checks can be triggered on a matched node, be careful when running this without a filter as it will
trigger all nodes concurrently.

```nohighlight
$ choria req scout trigger check=mailq -I dev1.example.net
```

The `check` argument is optional, when not given all checks will be triggered.

### Maintenance

Checks may be put into maintenance mode, they will not change state or be regularly checked until resumed.  We will
in future support a timeout setting which will auto resume checks after a period.

```nohighlight
$ choria req scout maintenance check=mailq -I dev1.example.net
```

Checks can be resumed later:

```nohighlight
$ choria req scout resume check=mailq -I dev1.example.net
```

The `check` argument is optional, when not given all checks will be affected

### Invoking Goss validations

We support a `goss` builtin for running a specific goss validation regularly, but we also support adhoc validations
via the Node API:

```nohighlight
$ choria req goss_validate file=/etc/goss.yaml vars=/etc/goss-vars.yaml
```

Here we invoke goss against `/etc/goss.yaml` with variables loaded from `/etc/goss-vars.yaml`, both these files must
exit on the node already.

In time we will add support for sending files as a byte stream from a central location for truely adhoc validations.

## Go API

We have a Golang API that can be used to create custom automation tools to manage Scout checks, here's an example
using it to trigger a check.

```go
package main

import (
	"context"
	"fmt"

	scoutagent "github.com/choria-io/go-choria/scout/agent/scout"
	scoutclient "github.com/choria-io/go-choria/scout/client/scout"
)

func main() {
	scout := scoutclient.Must()
	scout.OptionIdentityFilter("dev1.example.net")

	res, err := scout.Trigger().Checks([]interface{}{"mailq"}).Do(context.Background())
	if err != nil {
		panic(err)
	}

	res.EachOutput(func(r *scoutclient.TriggerOutput) {
		data := &scoutagent.TriggerReply{}
		err = r.ParseTriggerOutput(data)
		if err != nil {
			fmt.Printf("Invalid result from: %s: %s\n", r.ResultDetails().Sender(), err)
			return
		}

		fmt.Printf("%s:\n", r.ResultDetails().Sender())
		fmt.Printf("\tTransitioned: %v\n", data.TransitionedChecks)
		fmt.Printf("\tSkipped: %v\n", data.SkippedChecks)
		fmt.Printf("\tFailed: %v\n", data.FailedChecks)
	})
}
```

This is the equivalent of _choria req scout trigger check=mailq -I dev1.example.net_.

When run this produce the following output:

```nohighlight
$ ~/trigger
dev1.example.net:
        Transitioned: [mailq]
        Skipped: []
        Failed: []
```
