+++
title = "Go Clients"
weight = 11
+++

We have a [Golang RPC library](https://godoc.org/github.com/choria-io/mcorpc-agent-provider/mcorpc/client) that's similar in spirit to the Ruby client library while being more idiomatic Golang and more suitable to long running large scale automation tasks.  Use this if you want to write some form of long running never ending automated system or scale to very large fleets. This library can interact with any Choria RPC Agent based on their DDL, you do not need to generate any code or stubs. A very good example of this library in use is the code for the [choria req](https://github.com/choria-io/go-choria/blob/master/cmd/req.go) utility.

By it's nature it's more verbose and more involved to use - while the Ruby one is optimized for short quick scripts.

Recently we added the ability to generate focussed clients for Choria RPC Agents that are very easy to use and yields quite easy to read code. These generated clients use the above mentioned client library so they are scalable to huge fleets and suitable for use in lond running orchestration systems.

This guide will focus on these generated clients. You're encouraged to consider them first when looking at interacting with your fleet from Go.

## Generated Clients

As you might be aware every Agent has a DDL file that describes the agent - all it's actions, inputs, outputs, aggregation methods and validations. As of a recent improvement these DDL files now contain enough that highly usable clients can be generated for static languages like Golang.

{{% notice tip %}}
This capability is available as of version *0.14.0* of Choria
{{% /notice %}}

## Preparing your DDL

The generator will handle almost all existing agent DDLs, however in the past we did not support or enforce data types for the output items from agents. This makes it extremely hard to create fully usable clients for static languages like Golang.

We've recently added optional type hints to outputs in DDLs and I strongly suggest you take a look at the DDL files you're intending to use and add output hints. You should also review the code and ensure that the output types don't vary, if an item vary and you cannot fix that scenario then leave it untyped so Go will give you *interface{}* instances which you can then handle via *reflect*.

Here's a sample fixup I did of the class [puppet agent](https://github.com/choria-plugins/puppet-agent/pull/41/files) as an example, it's easier than it sounds!

## Generating the client

We'll look at building a custom utility to do what `mco rpc puppet disable message="testing golang" -W customer=acme` does, but using the new Go generated client, we'll also show how to do things like batching and finally custom discovery.

```nohighlight
$ mkdir -p puppet/disable
$ mkdir -p puppet/client
$ cd puppet
$ choria tool generate client /.../puppet.json client
INFO[0000] Writing Choria Client for Agent puppet Version 2.3.2 to client
INFO[0000] Writing client/action_disable.go for action disable
INFO[0000] Writing client/action_enable.go for action enable
INFO[0000] Writing client/action_last_run_summary.go for action last_run_summary
INFO[0000] Writing client/action_resource.go for action resource
INFO[0000] Writing client/action_runonce.go for action runonce
INFO[0000] Writing client/action_status.go for action status
INFO[0000] Writing client/resultdetails.go
INFO[0000] Writing client/requestor.go
INFO[0000] Writing client/ddl.go
INFO[0000] Writing client/discover.go
INFO[0000] Writing client/rpcoptions.go
INFO[0000] Writing client/client.go
INFO[0000] Writing client/initoptions.go
INFO[0000] Writing client/logging.go
INFO[0000] Writing client/doc.go
```

Here the */.../puppet.json* is a path to the DDL, often this would be in */opt/puppetlabs/mcollective/plugins/mcollective/agent/puppet.json*.

This creates the client, lets initialize go mod:

```nohighlight
$ go mod init example.net/puppet
$ go mod tidy
$ cat go.mod
module example.net/puppet

go 1.13

require (
        github.com/choria-io/go-choria v0.13.0
        github.com/choria-io/go-client v0.5.2
        github.com/choria-io/go-config v0.0.5
        github.com/choria-io/go-protocol v1.3.2
        github.com/choria-io/go-srvcache v0.0.6
        github.com/choria-io/mcorpc-agent-provider v0.9.0
        github.com/sirupsen/logrus v1.4.2
)
```

## Writing a *puppet disable* tool

The generated API has methods for every action, every input and output, allows you to write custom discovery plugins and more.

Here's a basic *disable* tool, while this has some niceties missing like progress bars and custom configs it shows that setting up and getting going is really easy and even this code will comfortably scale to 50 000 nodes or more:

```golang
package main

import (
	"context"
	"fmt"

	p "example.net/puppet/client"
)

func panicIfErr(err error) {
    if err != nil {
        panic(err)
    }
}

func disable(ctx context.Context) error {
    // instance of puppet client, panics if fails
    // uses default choria config path as setup by the modules
    pc := p.Must()

    // disables puppet with a custom message and custom filter
    res, err := pc.OptionFactFilter("customer=acme").Disable().Message("testing golang").Do(ctx)
    panicIfErr(err)

    res.EachOutput(func(r *p.DisableOutput) {
        if r.ResultDetails().OK() {
            fmt.Printf("OK: %-40s: enabled: %v\n", r.ResultDetails().Sender(), r.Enabled())
        } else {
            fmt.Printf("!!: %-40s: message: %s\n", r.ResultDetails().Sender(), r.ResultDetails().StatusMessage())
        }
    })

    nr := res.Stats().NoResponseFrom()
    if len(nr) > 0 {
        fmt.Printf("No responses received from %d hosts", len(nr))
    }

    return nil
}

func main() {
    panicIfErr(disable(context.Background()))
}
```

## Code Details

Lets look at a few key items about this code.

### Setup

Here we do the basic setup, config loading, finding the network, security etc. In this case I am panicing on any error:

```golang
pc := p.Must()
```

We can also pass in various options and do more traditional error handling.  Some other options are *Logger()* and *Discovery()* which will show later.

```golang
pc, err := p.New(p.ConfigFile("~/.puppet_client.conf"))
```

### Batching, canaries, discovery filters etc

In the example we have this:

```golang
res, err := pc.OptionFactFilter("customer=acme").Disable().Message("testing golang").Do(ctx)
```

Here the *OptionFactFilter()* is a RPC framework option thats applicable to any RPC call and not related to any specific agent.

The godoc comments it the definitive document, but here are a few of the options you'd have:

|Option|Description|
|------|-----------|
|`OptionReset()`|Put this first to reset all the options from previous calls else they are sticky|
|`OptionFactFilter(...string)`|One or more fact filters, matches the behavior of *-F* on the CLI|
|`OptionCollective(string)`|The name of the sub collective to target, matches *-T* on the CLI|
|`OptionInBatches(size, sleep int)`|Performs the task in batches with a specific sleep, *--batch* and *--batch-sleep* on the CLI|
|`OptionDiscoveryTimeout(time.Duration)`|How long to wait for discovery, matched *--discovery-timeout* or *--dt* on the CLI|
|`OptionLimitSize(string)`|Limit the request to a subset of nodes like *10* or *20%*, matches *--limit* on the CLI|
|`OptionLimitMethod(string)`|How to pick the random set *random* or *first*, no CLI equivalent but settable in the config|
|`OptionLimitSeed(int64)`|When using *random* method this lets you initialize the random number, set to the same number for predictable select|

### Actions and Inputs

In the example we have this:

```golang
res, err := pc.OptionFactFilter("customer=acme").Disable().Message("testing golang").Do(ctx)
```

Here we are invoking the *Disable()* action, it has no mandatory inputs.  If it for example *message* was required then the function would have been *Disable(message string)*.  In this case though *message* is optional thus you call *Disable().Message("optional message")*, you can chain in all other optional inputs in the same manner.

### Outputs

The *disable* action has 2 outputs - *enabled* and *status*.  In the Puppet Agent DDL I specified their data types so the method signatures are:

```golang
func (d *DisableOutput) Enabled() bool
func (d *DisableOutput) Status() string
```

If one of these did not have type hints the return type would have been *interface{}* and you'd need to figure that out yourself. Some types like *hash* can not be turned into structures automatically so they would be *map[string]interface{}*.

### Results

Here we iterate all the results and show a small one liner:

```golang
res, _ := pc.OptionFactFilter("customer=acme").Disable().Message("testing golang").Do(ctx)

res.EachOutput(func(r *p.DisableOutput) {
    if r.ResultDetails().OK() {
        fmt.Printf("OK: %-40s: enabled: %v\n", r.ResultDetails().Sender(), r.Enabled())
    } else {
        fmt.Printf("!!: %-40s: message: %s\n", r.ResultDetails().Sender(), r.ResultDetails().StatusMessage())
    }
})
```

The *ResultDetails()* gives you access to *Sender() string*, *OK() bool*, *StatusMessage() string* and *StatusCode() StatusCode*, these map to the similar things in the standard RPC libraries.

Finally you can call *res.HashMap()* which will give you a *map[string]interface{}* of the whole result structure.

### Statistics

At the end of the call the result will hold some statistics, here we show nodes that did not respond:

```golang
nr := res.Stats().NoResponseFrom()
if len(nr) > 0 {
    fmt.Printf("No responses received from %d hosts", len(nr))
}
```

A wealth of information is available, the interface for the stats can be seen here:

```golang
type Stats interface {
    Agent() string
    Action() string
    All() bool
    NoResponseFrom() []string
    UnexpectedResponseFrom() []string
    DiscoveredCount() int
    DiscoveredNodes() *[]string
    FailCount() int
    OKCount() int
    ResponsesCount() int
    PublishDuration() (time.Duration, error)
    RequestDuration() (time.Duration, error)
    DiscoveryDuration() (time.Duration, error)
}
```

## Custom Discovery

By default the client uses a traditional broadcast discovery and that's the only method we supply, you can though integrate with your own easily.  Lets read a flat file.

```golang
type FlatFileDiscovery struct {
    nodesFile string
}

// Reset is there to clear caches, we don't cache here so noop
func (f *FlatFileDiscovery) Reset() {}

func (f *FlatFileDiscovery) Discover(ctx context.Context, _ ChoriaFramework, _ []FilterFunc) ([]string, error) {
    file, err := os.Open(f.nodesFile)
    if err != nil {
        return []string{}, err
    }
    defer file.Close()

    found := []string{}
    scanner := bufio.NewScanner(file)
    for scanner.Scan() {
        found = append(found, strings.TrimSpace(scanner.Text()))
    }

    err = scanner.Err()
    if err != nil {
        return []string{}, err
    }

    return found, nil
}
```

We can ask the client to use this discovery method like here:

```golang
pc, err := p.New(p.Discovery(&FlatFileDiscovery{"/home/you/nodes.txt"}))
```

Now instead of network discovery the file will be read instead.
