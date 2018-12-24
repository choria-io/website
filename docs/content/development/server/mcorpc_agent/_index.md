+++
title = "MCO RPC Agent"
weight = 20
+++

Agents can be written that is compatible with the Ruby MCollective RPC API - covered separately in this section - here I'll show a very basic echo agent and how to plug it into Choria Server at compile time.

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

func New(mgr server.AgentManager) (*mcorpc.Agent, error) {
    metadata := &agents.Metadata{
		Name:        "echo",
		Description: "Choria Echo Agent",
		Author:      "R.I.Pienaar <rip@devco.net>",
		Version:     "1.0.0",
		License:     "Apache-2",
		Timeout:     2,
		URL:         "http://choria.io",
	}

    agent := mcorpc.New("echo", metadata, mgr.Choria(), mgr.Logger())

    agent.MustRegisterAction("ping", pingAction)
})

func echoAction(ctx context.Context, req *mcorpc.Request, reply *mcorpc.Reply, agent *mcorpc.Agent, conn choria.ConnectorInfo) {
	i := &EchoRequest{}
	if !mcorpc.ParseRequestData(i, req, reply) {
		// reply was already filled in with appropriate error messages by ParseRequestData
		return
	}

	reply.Data = &EchoReply{i.Message}
}
```

You'll have to write a DDL file like for Ruby agents, review the MCO RPC section in the docs about that, here we're mainly covering compiling this into the server.

Next we have to implement the *plugin.Pluggable* interface, there's a convenient helper to do this for you, lets see how *plugin.go* of the agent would look:

```golang
package echo

import (
	"github.com/choria-io/go-choria/plugin"
	"github.com/choria-io/mcorpc-agent-provider/mcorpc"
)

// ChoriaPlugin produces the plugin for choria
func ChoriaPlugin() plugin.Pluggable {
	return mcorpc.NewChoriaAgentPlugin(Metadata, New)
}
```