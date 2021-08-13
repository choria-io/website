+++
title = "Provisioning Target"
weight = 50
+++

Choria Server supports a Provisioning Mode where it will start up with a specific configuration allowing a piece of software to dynamically configure it. You can read more about this in the [Provisioning Agent](https://github.com/choria-io/provisioner) repository we also have a [video](https://youtu.be/7sGHf55_OQM) explaining the concept and message flow.

The problem with provisioning is that you have to connect somewhere where there is a provisioner. You can imagine, for example, if you have 30 data centers you might want to provision regionally to each DC. You could make it so that you always arrange for lets say *choria-provision.$dc.example.net* resolving as *choria-provision* but what if you have many different kinds of network or do not have the ability to influence DNS in this manner? A simple static build time configuration does not work.

Choria therefore lets you plug your own logic into it where you can resolve things however you wish.  You can do FQDN parsing, or call out to your cloud provider API, or speak to something like Consul or etcd to do service discovery. This plugin is called a *Provisioning Target* and we'll show you how to build a basic one.

By default, we ship a Provisioner Target that uses a JWT file, you deploy this file with your server and it provides all the hints needed for provisioning.  In generally I strongly recommend that is the model to use rather than write your own targets.

## File based hints

Let's write a Provisioning Target that reads */etc/dc.json* and construct a provisioning name using that information:

The interface you need to implement is *provtarget.TargetResolver*, it looks like this ([godoc](https://godoc.org/github.com/choria-io/go-choria/provtarget)):

```go
// TargetResolver is capable of resolving the target brokers for provisioning into lists of strings in the format host:port
type TargetResolver interface {
	// Name the display name that will be shown in places like `choria buildinfo`
	Name() string

	// Targets will be called to determine the provisioning destination
	Targets(context.Context, *logrus.Entry) []string
}
```

That's pretty easy, let's write a small provider.

It's important that these resolvers should retry more or less forever - or perhaps fall back to a static value.  Any number of things can prevent this from working and you would not want your unprovisioned servers to exit and remain unprovisioned. So we retry forever with a backoff based sleep between tries.

```go
package acme

import (
	"github.com/choria-io/go-choria/backoff"

	"github.com/sirupsen/logrus"
)

// DCHint is the data to put in the JSON file
type DCHint struct {
	DC string `json:"datacenter"`
}

type HintResolver struct{}

// Name implements provtarget.TargetResolver
func (r *HintResolver) Name() string {
	return "Acme DC hints Provisioning Target"
}

func (r *HintResolver) Targets(ctx context.Context, log *logrus.Entry) (targets []string) {
	try := 1

	for {
		// check the context before starting a try
		if ctx.Err() != nil {
			return targets
		}

		targets, err := resolve(log)
		if err != nil {
			log.Errorf("Could not resolve provisioning targets: %s", err)
		} else if len(targets) > 0 {
			return targets
		}

		// sleep in a way that the context can interrupt and try again
		log.Infof("Sleeping %s before trying to resolve provisioning target again", backoff.FiveSec.Duration(try))
		backoff.FiveSec.InterruptableSleep(ctx, try)

		try++
	}
}

func resolve(log *logrus.Entry) (targets []string, err error) {
	d, err := ioutil.ReadFile("/etc/dc.json")
	if err != nil {
		return targets, err
	}

	hint := &DCHint{}
	err = json.Unmarshal(d, hint)
	if err != nil {
		return targets, err
	}

	if hint.DC == "" {
		return targets, errors.New("No DC hint were provided in /etc/dc.json")
	}

	return []string{fmt.Sprintf("_provision.%s.example.net", hint.DC)}, nil
}
```

We need to provide the usual *plugin.Pluggable* so let's add that:

```go
package acme

import (
	"github.com/choria-io/go-choria/plugin"
)

// PluginInstance implements plugin.Pluggable
func (r *HintResolver) PluginInstance() interface{} {
	return r
}

// PluginVersion implements plugin.Pluggable
func (r *HintResolver) PluginVersion() string {
	return "1.0.0"
}

// PluginName implements plugin.Pluggable
func (r *HintResolver) PluginName() string {
	return r.Name() + " version " + r.PluginVersion()
}

// PluginType implements plugin.Pluggable
func (r *HintResolver) PluginType() plugin.Type {
	return plugin.ProvisionTargetResolverPlugin
}

// ChoriaPlugin produces the plugin for choria
func ChoriaPlugin() plugin.Pluggable {
	return &HintResolver{}
}
```

## Compile into Choria

You can now build your own Choria instance based on the [Packaging](../packaging) documentation, you'll load your plugin as follows in the `packager/user_plugins.yaml`

```yaml
---
dc_hint_provtarget: gitlab.example.net/ops/dc_hint_provtarget
```

Once built the `choria buildinfo` command will show you that you have yours enabled:

```bash
$ choria buildinfo
...
    Provisioning Target Resolver: Acme DC hints Provisioning Target version 1.0.0
...
```
