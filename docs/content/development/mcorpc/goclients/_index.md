+++
title = "Golang Clients"
weight = 11
+++

In addition to the [Ruby Client](../client/) there is a new Golang client being developed.  Today it's quite feature complete and implements 90% of the features the Ruby one does.

By its nature it's more verbose but it's more tailored for robust long running automation systems than the Ruby one was, less about 1 liners and quick scripts and more about high performance and stability in the long term wrt memory usage and CPU consumption.

We'll walk through building a client application showing how to accomplish common tasks, the [Godoc](https://godoc.org/github.com/choria-io/mcorpc-agent-provider/mcorpc/client) should be considered a companion to this.

## The Choria Framework

All these client interactions require you to have a instance of the [Choria Framework](https://godoc.org/github.com/choria-io/go-choria/choria) setup.

```golang
fw, err := choria.New(choria.UserConfig())
panicIfErr(err)
```

This loads the user configuration, handles creation of connection to the network and so forth. You won't interact with it a lot but the RPC client wants it.

## JSON DDLs

The Golang Client supports DDL files but they have to be in JSON format. All the packaged agents automatically convert their DDL to JSON, there is a [schema](choria.io/schemas/mcorpc/ddl/v1/agent.json) describing a valid DDL file.

In general the client will find it's own but you can also supply your own if you load it from elsewhere.  I find embedding them in my Go application is quite nice to ensure there are no dependencies.  In that case you can pass it to the client, more below.

## Creating a client instance

All the example below will need a client instance, here's a few ways to create it - we're creating an instance to the [Puppet Agent](https://github.com/choria-plugins/puppet-agent):

```golang
import (
    rpcclient "github.com/choria-io/mcorpc-agent-provider/mcorpc/client"
)

func req() {
    fw, err := choria.New(choria.UserConfig())
    panicIfErr(err)

    puppet, err := rpcclient.New(fw, "puppet")
    panicIfErr(err)
}
```
{{% notice tip %}}
Past this point we won't include boiler plate like imports etc, your IDE should do most of it for you anyway
{{% /notice %}}

## Call an Agent and print the result

```golang
ctx, cancel := context.WithCancel(context.Backgroun())
defer cancel

handler := func(pr protocol.Reply, reply *rpc.RPCReply) {
    fmt.Printf("%s: %s\n", )
}

result, err := client.Do(ctx, "echo", map[string]interface{}{"msg": "Welcome to Golang RPC"})
```

This invokes the `echo` action on the `helloworld` agent with the input `msg` set to `Welcome to Golang RPC`.

