+++
title = "Data Stores"
weight = 450
toc = true
+++

Data Stores allow you to dynamically read and write data from tools like Consul, etcd or a local memory space.

Data Stores are used in a number of places:

  * Inputs can be dynamically read from data stores
  * Data can be written to the data stores
  * Data can be deleted from the data stores
  * Playbooks can obtain a playbook level lock against a data store ensuring single runs at a time

Planned features are:

  * Task level locks
  * Service membership for node sets

### Defining a Data Store

You define a data store with a unique name and store specific properties, you can then reference it in inputs, tasks etc by name.

#### Common Data Store Properties

```yaml
data_stores:
  local_consul:
    type: consul
    timeout: 120
    ttl: 60
```

This creates a data store called *local_consul*.

|Option|Description|
|------|-----------|
|type|Tye type of data store, like *memory* or *consul*, see below for more|
|timeout|How long to wait for a lock to be obtained|
|ttl|How long a lock should be valid before expiring, this protects against stale locks if a playbook exits early|

{{% notice tip %}}
You can use templates when defining Data Store so you can use inputs to vary hostnames, usernames etc
{{% /notice %}}

#### Memory Store

```yaml
data_stores:
  pb_memory:
    type: "memory"
```

The memory store is a purely in-memory store and lives only for the length of the playbook run.  As such when you use it for locks these locks are not distributed or networked.

It has no options other than the basic ones that apply to all stores.

#### Environment Store

```yaml
data_stores:
  pb_env:
    type: "environment"
    prefix: "PB_"
```

The environment store reads and writes variables from your shell environment.  It supports read, write and delete, no locking.

|Option|Description|
|------|-----------|
|prefix|Setting a prefix of *PB_* will fetch environment variable *PB_test* when requesting *test*.|

#### File Store

```yaml
data_stores:
  pb_yaml:
    type: "file"
    file: "~/pb_data.yaml"
    format: "yaml"
```

The file store reads and writes variables to a file on your local machine.  It supports read, write and delete, no locking.

|Option|Description|
|------|-----------|
|file|The file to store data in. The file has to exist, but it can be 0 bytes at start.|
|format|The file format to use, *yaml* and *json* are valid values|

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

|Option|Description|
|------|-----------|
|ttl|How long locks should be valid for after playbook crash or similar. Locks will be refreshed 5 seconds before expiry.  10 seconds minimum|
|timeout|How long to wait for a lock, fails after timeout|

### Shell Store

{{% notice tip %}}
This feature is included since *0.0.23*
{{% /notice %}}

```yaml
data_stores:
  pb_shell:
    type: "shell"
    command: "/path/to/store.sh"
    timeout: 20
    cwd: "/path/to"
    environment:
      "EXAMPLE": "VALUE"
```

This type of store exist to make it easy for users to integrate existing data stores into the playbooks using just a shell command.

The options are:

|Option|Description|
|------|-----------|
|command|Path to the command that will be run.  If you pass any inputs to this script make sure to validate them as *shellsafe*|
|timeout|How long the command is allowed to run before it's killed, defaults to *10*|
|cwd|The working directory of the command, defaults to your temporary directory|
|environment|Any environment variables you wish to set, all have to be strings|

The design of this is to make it easy for you to write commands in any language.  Valid keys have to match */^\[a-zA-Z0-9_-\]+$/*

#### Reads
When data is requested for reading *key* your command is run as */path/to/store.sh --read key*

Command environment will have:

|Option|Description|
|------|-----------|
|CHORIA_DATA_KEY|The key to read|
|CHORIA_DATA_ACTION|read|

You should return the value in a single line to STDOUT, STDERR is ignored. Exiting non zero is failure.

#### Writes
When data is requested for writing to *key* your command is run as */path/to/store.sh --write key*

Command environment will have:

|Option|Description|
|------|-----------|
|CHORIA_DATA_KEY|The key to write|
|CHORIA_DATA_ACTION|write|
|CHORIA_DATA_VALUE|The value to be written|

All output is ignored. Exiting non zero is failure.

#### Deletes
When you request deletion of *key* your command is run as */path/to/store.sh --delete key*

Command environment will have:

|Option|Description|
|------|-----------|
|CHORIA_DATA_KEY|The key to delete|
|CHORIA_DATA_ACTION|delete|

All output is ignored. Exiting non zero is failure.

### Binding inputs to Data Stores

When you define an input and add the *data* key to it this should reference a previously defined data store.

```yaml
inputs:
  cluster:
    description: "Cluster to deploy"
    type: "String"
    default: "alpha"
    validation: ":shellsafe"
    data: "local_consul/cluster"
```

You'll still be able to supply the input on the CLI - in which case it becomes static and does not change for the life of the playbook - but if you do not it will bind to the key *cluster* in the data store called *local_consul*.  If you add the key *dynamic* and set it to true then the input will not appear on the CLI and will only resolve from data stores and defaults

From then on any time you reference it in a template like *{{{ inputs.cluster }}}* it will query the data store and will not cache this result.  So if the playbook or an extenal tool adjusts the data in the data store the playbook will always get current data.

For the moment only String data is supported by the data store, but validation will happen like *:shellsafe* here.

### Playbook level locks

You might want to ensure a playbook is only ever run once, you can add a playbook level lock which when used against something like Consul will ensure a playbook is only run once.

```yaml
locks:
  - consul_store
  - consul_store/specific_lock
```

These locks will be obtained early in the playbook startup and if they can't the playbook will fail without producing a report.

There will be timeout for how long at most it will wait to get a lock, set using the *timeout* option when creating the data store.

When it makes sense the data store will create the lock in a way that should the playbook die, machine dies or other unexpected thing the lock will timeout after a period configurable using the *ttl* option when creating the data store.

{{% notice tip %}}
Locks like these make no sense with the *memory* data store, you need a service like Consul
{{% /notice %}}

When you just specify a data store name like *consul_store* the lock will match the name property of the playbook else you can specify your own path like *consul_store/specific_lock*.

### Editing Data Stores

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
