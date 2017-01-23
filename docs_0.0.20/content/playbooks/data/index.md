+++
title = "Data Sources"
weight = 450
toc = true
+++

Data Sources allow you to dynamically read and write data from tools like Consul, etcd or a local memory space.

Data Sources are used in a number of places:

  * Inputs can be dynamically read from data stores
  * Data can be written to the data stores
  * Data can be deleted from the data stores
  * Playbooks can obtain a playbook level lock against a data store ensuring single runs at a time

Planned features are:

  * Task level locks
  * Service membership for node sets

{{% notice tip %}}
This feature is included since *0.0.20*
{{% /notice %}}

### Defining a Data Source

You define a data source with a unique name and source specific properties, you can then reference it in inputs, tasks etc by name.

#### Common Data Source Properties

```yaml
data_sources:
  local_memory:
    type: memory
    timeout: 120
    ttl: 60
```

This creates a data source called *local_memory*, for the moment only a basic local to the playbook memory store exist.

|Option|Description|
|------|-----------|
|type|Tye type of data store, only *memory* is currently supported|
|timeout|How long to wait for a lock to be obtained|
|ttl|How long a lock should be valid before expiring, this protects against stale locks if a playbook exits early|

{{% notice tip %}}
You can use templates when defining Data Sources so you can use inputs to vary hostnames, usernames etc
{{% /notice %}}

#### Memory Store

```yaml
data_stores:
  pb_memory:
    type: "memory"
```

The memory store is a purely in-memory store and lives only for the length of the playbook run.  As such when you use it for locks these locks are not distributed or networked.

It has no options other than the basic ones that apply to all stores.

#### Consul Store

```yaml
data_stores:
  pb_consul:
    type: "consul"
    ttl: 20
    timeout: 120

locks:
  - pb_consul
```

To use the Consul store you should have the `diplomat` gem installed in your Puppet Ruby on the node where you will run *mco playbook*.  You can use Hiera to do this:

```yaml
mcollective_choria::gem_dependencies:
  "diplomat": "1.1.0"
```

The Consul store requires you to have a local [Consul Agent](https://consul.io) instance running on your node where you run *mco playbook* from.  Configuring a Consul cluster is out of scope for this guide.

Using this Data Store you can obtain network wide excludive locks that ensure a specific playbook is run once only and you can store, read and delete data in the Consul store.

When using locks a Session is created and maintained, should the playbook die unexpectedly the Session will expire after *ttl* seconds.  In the example above a lock will be created in Consul called *choria/locks/playbook/playbook_name*.

Should a lock not be obtained after *timeout* seconds the playbook will fail without writing a report.

### Binding inputs to Data Sources

When you define an input and add the *data* key to it this should reference a previously defined data source.

```yaml
inputs:
  cluster:
    description: "Cluster to deploy"
    type: "String"
    default: "alpha"
    validation: ":shellsafe"
    data: "local_memory/cluster"
```

You'll still be able to supply the input on the CLI - in which case it becomes static and does not change for the life of the playbook - but if you do not it will bind to the key *cluster* in the data store called *local_memory*.  If you add the key *dynamic* and set it to true then the input will not appear on the CLI and will only resolve from data sources and defaults

From then on any time you reference it in a template like *{{{ inputs.cluster }}}* it will query the data store and will not cache this result.  So if the playbook or an extenal tool adjusts the data in the data source the playbook will always get current data.

For the moment only String data is supported by the data sources, but validation will happen like *:shellsafe* here.

### Playbook level locks

You might want to ensure a playbook is only ever run once, you can add a playbook level lock which when used against something like Consul will ensure a playbook is only run once.

```yaml
locks:
  - consul_store
  - consul_store/specific_lock
```

These locks will be obtained early in the playbook startup and if they can't the playbook will fail without producing a report.

There will be timeout for how long at most it will wait to get a lock, set using the *timeout* option when creating the data store.

When it makes sense the data source will create the lock in a way that should the playbook die, machine dies or other unexpected thing the lock will timeout after a period configurable using the *ttl* option when creating the data store.

{{% notice tip %}}
Locks like these make no sense with the *memory* data source, you need a service like Consul
{{% /notice %}}

When you just specify a data source name like *consul_store* the lock will match the name property of the playbook else you can specify your own path like *consul_store/specific_lock*.

### Editing Data Sources

A playbook can write and delete data using the new *data* task - see the [Tasks](../tasks/) reference for full details.

```yaml
inputs:
  cluster:
    description: "Cluster to deploy"
    type: "String"
    dynamic: true
    data: "pb_consul/choria/kv/cluster"
    default: "alpha"

tasks:
  - shell:
      command: "/usr/local/bin/deploy --cluster {{{ inputs.cluster }}}"

  - data:
      action: "write"
      key: "pb_consul/choria/kv/cluster"
      value: "bravo"

  - shell:
      command: "/usr/local/bin/deploy --cluster {{{ inputs.cluster }}}"

hooks:
  pre_book:
    - data:
        action: "delete"
        key: "pb_consul/choria/kv/cluster"
```

Here we have a *dynamic* input bound to Consul, we ensure it's empty on startup so it will use the default *alpha*, we run our deploy task(s) and then update the data to the next cluster and deploy that.

Put in a few extra steps between the two to validate your deploys against the *alpha* cluster and you can do a blue/green deploy in the same playbook.  The 2 shell tasks will have a different value for the *cluster* input.

### Reports

Your Reports will include both *dynamic* and *static* inputs.  In the case of dynamic ones you will only get a list of the inputs that were dynamic and not their data since the data cannot be known and have different values for the life of the playbook.
