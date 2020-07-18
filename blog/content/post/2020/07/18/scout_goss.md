---
title: "Scout Goss Integration"
date: 2020-07-18T09:17:00+01:00
tags: ["scout"]
draft: false
---

In the [Scout Announcement](https://choria.io/blog/post/2020/07/02/choria_scout/) blog post I mentioned we are looking 
to integrate [Goss](https://github.com/aelsabbahy/goss) into Scout and I wanted to post an update on that.

## Background

Goss is something similar to `serverspec` - it lets you write unit tests about your nodes actual state rather than code used to build it. 
Goss definitions are written in YAML or JSON and supports Go templating for customization.

This model is well suited for the purposes of monitoring since you can write really in depth sets of validations and treat them as a single unit.

Goss is written in Go, very fast and thanks to a lot of work I did recently embeddable in other software.

Here's an example Goss specification:

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

You can see we are able to check many types of server resource and combine them into one test. If we combined this with remediation
and the ability to run this continuously or adhoc we can have a nice framework to build something powerful.

## Scout Integration

Today Scout is configurable from within Puppet, in our next release which should come next week, we will release Goss support.

I've set things up so you can use the Hierarchical data from Hiera to create your Goss specification:

```yaml
choria::scout_gossfile:
  package:
    openssh-server:
      installed: true
  port:
    tcp:22:
      listening: true
      ip:
      - 0.0.0.0
  service:
    sshd:
      enabled: true
      running: true
```

This is automatically deep merged in Puppet - you can have layers in your hierarchy contribute checks to have unique sets of checks for various parts of your fleet.

With this done, we can now schedule a regular check using goss:

```puppet
choria::scout_check{"goss":
    builtin => "goss"
}
```

This will do a full validate every 5 minutes and produce CloudEvents to the Choria network.  Failures will map to `CRITICAL`
so these will behave like any other Nagios like check.  No extra dependencies are needed. It supports remediation hooks
and all the other usual check settings.

When running as a check the node overrides file is used as Goss Variables, meaning you can do data interpolation using templates.

We do not currently publish the full reports to the network, but that's something we an look at in the future.

```nohighlight
OK: OK: Count: 5, Failed: 0, Duration: 0.067s|checks=5;; failed=0;; runtime=0.067405s
```

## Adhoc tests

In our next release we will also ship a `scout` agent to all nodes, this can be used to trigger checks, set maintenance etc
but also to run adhoc Goss validations.

![goss_validate action](goss_action.png)

Here we validate a specific YAML file which should already exist, we will add the ability to send adhoc YAML to the nodes soon.

These runs return a lot of data about each check, not really usable on the CLI but as a API building block this will help us a lot.

## Golang API

Speaking of APIs, for the first time in Choria we are publishing Golang client packages to interact with a specific agent, here's
an example of retrieving the list of checks from a node.

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

You will be able to invoke Goss and interact with the results via the same API.

## Conclusion

I have some other ideas for Goss, but as a starting point I am really happy with where this is and glad to see the over
1 000 line patch I did to Goss finally being used.
