+++
title = "MCO RPC Agent"
weight = 20
+++

Agents can be written that are compatible with the Ruby MCollective RPC API - covered separately in this section - here I'll show a very basic echo agent and how to plug it into Choria Server at compile time.

## Echo Agent

Below a very basic agent that would respond to `mco rpc echo ping message="hello world"`.

```golang
package echo

type EchoRequest struct {
    Message string `json:"message" validate:"shellsafe"`
}

type EchoReply struct {
    Message string `json:"message"`
    TimeStamp int `json:"timestamp"`
}

var metadata = &agents.Metadata{
	Name:        "echo",
	Description: "Choria Echo Agent",
	Author:      "R.I.Pienaar <rip@devco.net>",
	Version:     "1.0.0",
	License:     "Apache-2",
	Timeout:     2,
	URL:         "http://choria.io",
}

func New(mgr server.AgentManager) (*mcorpc.Agent, error) {
	agent := mcorpc.New("echo", metadata, mgr.Choria(), mgr.Logger())

	agent.MustRegisterAction("ping", pingAction)

	return agent, err
})

func pingAction(ctx context.Context, req *mcorpc.Request, reply *mcorpc.Reply, agent *mcorpc.Agent, conn choria.ConnectorInfo) {
	i := &EchoRequest{}
	if !mcorpc.ParseRequestData(i, req, reply) {
		return
	}

	reply.Data = &EchoReply{i.Message}
}

// ChoriaPlugin produces the Choria pluggable plugin it uses the metadata
// to dynamically answer questions of name and version
func ChoriaPlugin() plugin.Pluggable {
	return mcorpc.NewChoriaAgentPlugin(metadata, New)
}
```

You'll have to write a DDL file like for Ruby agents, review the MCO RPC section in the docs about that, here we're mainly covering compiling this into the server.

Notice we show the `ChoriaPlugin()` function here that you saw in the [Plugin Interface](../plugin_interface) documentation, this is all you need to do as the framework will use the agent Metadata to supply values like Version, Name etc.

## Compile into Choria

You can now build your own Choria instance based on the [Packaging](../packaging) documentation, you'll load your plugin as follows in the `packager/user_plugins.yaml`

```yaml
---
echo_agent: gitlab.example.net/ops/echo_agent
```

Once built you'll be able to interact with the echo agent via any MCO RPC client - as long as you also write the DDL files.