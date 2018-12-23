+++
title = "Config Mutator"
weight = 30
+++

Configuration Mutator plugins allow you to dynamically adjust the configuration of the Choria Server. The server starts up with a number of hard coded defaults and build flags set during compilation and then reads the server.conf.

This is generally fine and I imagine you want to use this kind of plugin only in the extremely weird edge cases, examples below:

 * You want to enable/disable provisioning mode on criteria entirely unique to your setup
 * You want to dynamically enable/disable PKI or TLS security.  For instance some DCs might not have access to a CA.
 * You want to look up hosts to connect to using means other than SRV or static configuration
 * You want to prepare a token or other kind of credential, perhaps fetched from credential store during initialization

Mutators called after the server.conf is read and configuration defaults are set but before the framework initialize, so you have a chance to basically do any weird startup fiddling with the configuration or build settings.

These are all pretty weird edge cases, some of them might be better served by creating a specific kind of plugin in the Choria framework - like the token cases - but regardless you can use this to solve dynamic configuration cases pre server initialization.

Multiple mutators can be run in the same Choria instance.

## Dynamically control security settings

Lets look at a plugin that detects the presence of all the PKI related files and only enable PKI Security and TLS if everything is present. While I would not recommend this it would be helpful in a phased migration from a insecure to a secure network or allowing the same package to function in zones where you do not have or want a CA system.

### Mutator

```go
package acme

import (
	"os"

	"github.com/choria-io/go-choria/config"
	"github.com/choria-io/go-protocol/protocol"
	"github.com/sirupsen/logrus"
)

// Mutator configures choria to enable protocol security when all certs are present
type Mutator struct{}

// Mutate inspects the configuration for file security settings that configures
// paths, if those are set and a files all exist, are > 0 then protocol security
// is enabled otherwise the setting is left as is
func (c *Mutator) Mutate(cfg *config.Config, log *logrus.Entry) {
	// We only want to do this if we're running in file security mode, makes no sense in others
	if cfg.Choria.SecurityProvider != "file" {
		return
	}

	// We are already running in Secure mode, perhaps a compile time option, regardless lets not support
	// downgrading security - only upgrading security
	if protocol.IsSecure() {
		return
	}

	if cfg.Choria.FileSecurityCertificate == "" || cfg.Choria.FileSecurityKey == "" || cfg.Choria.FileSecurityCA == "" {
		log.Warn("Protocol security not enabled as all security files do not exist")
		return
	}

	if fileExistNonZero(cfg.Choria.FileSecurityCertificate) && fileExistNonZero(cfg.Choria.FileSecurityKey) && fileExistNonZero(cfg.Choria.FileSecurityCA) {
		log.Info("Enabling protocol security since all SSL configuration paths exist")

		// These adjust the build flags, they are always strings as they are setable from the CLI
		protocol.Secure = "true"
		build.TLS = "true"
	}
}

func fileExistNonZero(p string) bool {
	stat, err := os.Stat(p)
	if err != nil {
		return false
	}

	if stat.Size() > 0 {
		return true
	}

	return false
}
```

### Plugin

The plugin needs to have the *plugin.Pluggable* interface implemented, see the [Plugin Interface](../plugin_interface) documentation for the background on this:

```go
package acme

import (
	"github.com/choria-io/go-choria/plugin"
)

// PluginInstance implements plugin.Pluggable
func (c *Mutator) PluginInstance() interface{} {
	return c
}

// PluginVersion implements plugin.Pluggable
func (c *Mutator) PluginVersion() string {
	return "0.0.1"
}

// PluginName implements plugin.Pluggable
func (c *Mutator) PluginName() string {
	return "Acme Dynamic Protocol Security Configurer version " + c.PluginVersion()
}

// PluginType implements plugin.Pluggable
func (c *Mutator) PluginType() plugin.Type {
	return plugin.ConfigMutatorPlugin
}

// ChoriaPlugin produces the plugin for choria
func ChoriaPlugin() plugin.Pluggable {
	return &Mutator{}
}
```

### Tests

And finally we can add some tests to make sure all works:

```go
package acme

import (
	"io/ioutil"
	"testing"

	"github.com/choria-io/go-choria/config"
	"github.com/choria-io/go-protocol/protocol"
	"github.com/sirupsen/logrus"

	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
)

func TestFileContent(t *testing.T) {
	RegisterFailHandler(Fail)
	RunSpecs(t, "ConfigMutator/Acme")
}

var _ = Describe("configurator/acme", func() {
	It("Should only enable security when requirements are met", func() {
		c, _ := config.NewDefaultConfig()
		m := &Mutator{}
		logger := logrus.New()
		logger.Out = ioutil.Discard
		log := logrus.NewEntry(logger)

		protocol.Secure = "false"
		build.TLS = "false"

		c.Choria.SecurityProvider = "puppet"
		m.Mutate(c, log)
		Expect(protocol.IsSecure()).To(BeFalse())

		c.Choria.SecurityProvider = "file"
		m.Mutate(c, log)
		Expect(protocol.IsSecure()).To(BeFalse())

		c.Choria.FileSecurityCertificate = "testdata/zero.txt"
		c.Choria.FileSecurityKey = "testdata/zero.txt"
		c.Choria.FileSecurityCA = "testdata/zero.txt"
		m.Mutate(c, log)
		Expect(protocol.IsSecure()).To(BeFalse())

		Expect(fileExistNonZero("testdata/nonzero.txt")).To(BeTrue())

		c.Choria.FileSecurityCertificate = "testdata/nonzero.txt"
		c.Choria.FileSecurityKey = "testdata/nonzero.txt"
		c.Choria.FileSecurityCA = "testdata/nonzero.txt"
		m.Mutate(c, log)
		Expect(protocol.IsSecure()).To(BeTrue())
		Expect(build.HasTLS()).To(BeTrue())
	})
})
```

## Compile into Choria

You can now build your own Choria instance based on the [Packaging](../packaging) documentation, you'll load your plugin as follows in the `packager/user_plugins.yaml`

```yaml
---
dynamic_security: gitlab.example.net/ops/dynamic_security
```

Once built the output from `choria buildinfo` will show your mutator loaded.

```
$ choria buildinfo
...
Configuration Mutators:
       Acme Dynamic Protocol Security Configurer version 0.0.1
...
```