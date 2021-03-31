+++
title = "Choria Registration"
weight = 300
+++

Registration is the feature that lets each Choria Server publish key data about itself onto the network, the data being
published can vary based on your needs and, we have 2 plugins at the moment.

Registration tend to go hand-in-hand with Adapters since it is Adapters that receives, unpacks and republish the data 
published by Choria Server.

The model is where every node managed by Choria will on a frequency like 30 minutes publish their entire state, this means
the data is effectively standalone and loss in the storage layer will essentially immediately be repaired.

## Use Cases

We have used registration to great effect to build systems that does real time analysis of the overall shape of a fleet.

Imagine if you publish from registration data like what shared resources like a File server is being relied on by a given
node.  Using this data in a stream processing manner one can continuously monitor the overall multi node shape of the fleet
and highlight when for example 2 related clusters are accidentally configured to share the same Database and more.

One can also use the data to build asset databases by publishing all the facts about a node from the facts cache or use it
to create discovery sources by publishing all the internal inventory.

The use cases of this feature is very varied but, it's about making node level data available at a large scale to the rest
of the network.

## Registration Providers

Today Choria has only 2 Registration Data providers, the sections below document these providers and the configuration.

Generally speaking the data sent from Choria is encoded the same way a request is encoded, this means the data is signed 
by the node certificate and more.

While the current set of adapters do not verify these signatures they exist to allow strict control over the validity of
the published data and we might in time support validating the data against a CA for example.

### File Content Provider

The *file_content* provider reads the bytes of a file and publish it unmodified to the network. It has additional
features to track mtime of the file and inform you if the file has changed since last poll - and when it has changed.
Additionally, the data can be compressed across the wire.

Configuration is via Hiera:

```yaml
choria::server::config:
  # the type of provider to use
  registration: file_content

  # how frequently to send the data, in seconds
  registerinterval: 300 # the default
  
  # sleep a period between between 0 and registerinterval before the first publish
  registration_splay: true # the default
  
  # the file to publish
  plugin.choria.registration.file_content.data: /etc/inventory.yaml
  
  # a subject to publish it to
  plugin.choria.registration.file_content.target: choria.registration.%{facts.fqnd}
  
  # compress the data
  plugin.choria.registration.file_content.compression: true # the default  
```

Here we publish */etc/inventory.yaml* every 300 seconds after an initial sleep.

The data being published look like the Go struct below, and we publish a [JSON Schema](https://choria.io/schemas/choria/registration/v1/filecontent.json):

```go
type FileContentMessage struct {
	Mtime    int64  `json:"mtime"`
	File     string `json:"file"`
	Updated  bool   `json:"updated"`
	Protocol string `json:"protocol"`
	Content  []byte `json:"content,omitempty"`
	ZContent []byte `json:"zcontent,omitempty"`
}
```

|Key|Description|
|---|-----------|
|`mtime`|The unix time stamp the file was last modified|
|`file`|The path to the file being published|
|`update`|A hint that the file was modified since last poll, this is only a hint since we dont persist the state between restarts|
|`protocol`|Always `choria:registration:filecontent:1`|
|`content`|When compression is disabled the data will be in this key|
|`zcontent`|When compression is enabled the data will be in this key compressed using gzip|

### Inventory Content Provider

The *inventory_content* provider publish the internal state of the Choria Server, the published data includes facts, tagged classes, agent inventory and more.

Configuration is via Hiera:
```yaml
choria::server::config:
  registration: inventory_content
  registerinterval: 300 # the default
  registration_splay: true # the default

  # a subject to publish it to
  plugin.choria.registration.inventory_content.target: choria.registration.%{facts.fqnd}
  
  # compress the data
  plugin.choria.registration.inventory_content.compression: true # the default
```

The data being published looks like the Go struct below, and we publish a [JSON Schema](https://choria.io/schemas/choria/registration/v1/inventorycontent.json)

```go
type InventoryContentMessage struct {
	Protocol string          `json:"protocol"`
	Content  json.RawMessage `json:"content,omitempty"`
	ZContent []byte          `json:"zcontent,omitempty"`
}
```

|Key|Description|
|---|-----------|
|`protocol`|Always `choria:registration:inventorycontent:1`|
|`content`|When compression is disabled the data will be in this key|
|`zcontent`|When compression is enabled the data will be in this key compressed using gzip|

The data held in `content` or `zcontent` is a JSON document from the following structure:

```go
type InventoryData struct {
	Agents      []agents.Metadata          `json:"agents"`
	Classes     []string                   `json:"classes"`
	Facts       json.RawMessage            `json:"facts"`
	Status      *statistics.InstanceStatus `json:"status"`
	Collectives []string                   `json:"collectives"`
}
```

This is all the data needed to build a full discovery data source.
