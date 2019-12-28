+++
title = "Plugin Interface"
weight = 20
+++

All Choria plugins have to implement the same basic *plugin.Pluggable* interface that looks like this ([godoc](https://godoc.org/github.com/choria-io/go-choria/plugin)):

```go
// Pluggable is a Choria Plugin
type Pluggable interface {
	// PluginInstance is any structure that implements the plugin, should be right type for the kind of plugin
	PluginInstance() interface{}

	// PluginName is a human friendly name for the plugin
	PluginName() string

	// PluginType is the type of the plugin, to match plugin.Type
	PluginType() Type

	// PluginVersion is the version of the plugin
	PluginVersion() string
}
```

And you need a function in your package that produces an instance of the above interface:

```go
func ChoriaPlugin() plugin.Pluggable
```

Thus when you add your plugin to the plugin system like below in `packager/user_plugins.yaml`:

```yaml
---
myplugin: github.com/mycorp/myplugin
```

The system will call your *myplugin.ChoriaPlugin()* that should produce a *plugin.Pluggable*. An example of this can be found in the [Golang MCO RPC compatibility layer](https://godoc.org/github.com/choria-io/mcorpc-agent-provider/mcorpc/golang).

