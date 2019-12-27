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
  * Playbooks can wrap blocks of critical code in arbitrary locks to ensure only a single instance runs at a time

### Defining a Data Store

Here's a typical *consul* data store:

```puppet
$ds = {
  "type" => "consul",
  "timeout" => 120,
  "ttl" => 60
}
```

With the data source setup store, you can now read/write/delete data:

```puppet
# write
choria::data("api_token", "xx-secret-token", $ds)

# read it back later
$token = choria::data("api_token", $ds)
```

#### Common Data Store Properties

Here's a typical *consul* data store
```puppet
$ds = {
  "type" => "consul",
  "timeout" => 120,
  "ttl" => 60
}
```

This creates a data store called *local_consul*.

|Option|Description|
|------|-----------|
|type|Tye type of data store, like *memory* or *consul*, see below for more|
|timeout|How long to wait for a lock to be obtained|
|ttl|How long a lock should be valid before expiring, this protects against stale locks if a playbook exits early|

#### Environment Store

```puppet
$ds = {
  "type" => "environment",
  "prefix" => "PB_"
}
```

The environment store reads and writes variables from your shell environment.  It supports read, write and delete, no locking.

|Option|Description|
|------|-----------|
|prefix|Setting a prefix of *PB_* will fetch environment variable *PB_test* when requesting *test*.|

#### File Store

```puppet
$ds = {
  "type" => "file",
  "file" => "~/.playbook.rc",
  "format" => "yaml",
  "create" => true
}
```

The file store reads and writes variables to a file on your local machine.  It supports read, write and delete, no locking.

|Option|Description|
|------|-----------|
|file|The file to store data in. The file has to exist, but it can be 0 bytes at start.|
|format|The file format to use, *yaml* and *json* are valid values|
|create|By default and when this is false the data file has to exist, with true the file will be created|

#### Consul Store

```puppet
$ds = {
  "type" => "consul",
  "ttl" => 20,
  "timeout" => 120
}
```

To use the Consul store you should have the `diplomat` gem installed in your Puppet Ruby on the node where you will run *mco playbook*.  You can use Hiera to do this:

```yaml
mcollective_choria::gem_dependencies:
  "diplomat": "1.1.0"
```

The Consul store requires you to have a local [Consul Agent](https://consul.io) instance running on your node where you run *mco playbook* from.  Configuring a Consul cluster is out of scope for this guide.

Using this Data Store you can obtain network wide exclusive locks that ensure a specific playbook section is run once only and you can store, read and delete data in the Consul store.

When using locks a Session is created and maintained, should the playbook die unexpectedly the Session will expire after *ttl* seconds.

|Option|Description|
|------|-----------|
|ttl|How long locks should be valid for after playbook crash or similar. Locks will be refreshed 5 seconds before expiry.  10 seconds minimum|
|timeout|How long to wait for a lock, fails after timeout|

#### Etcd Store

```puppet
$ds = {
  "type" => "etcd",
  "url" => "http://etcd.example.net:2379",
  "user" => "admin",
  "password" => "secret"
}
```

To use the Etcd store you should have the `etcdv3` gem installed in your Puppet Ruby on the node where you will run *mco playbook*.  You can use Hiera to do this:

```yaml
mcollective_choria::gem_dependencies:
  "etcdv3": "0.6.0"
```

The Etcd store requires you to have a [Etcd Agent](https://coreos.com/etcd/docs/latest/) instance running.  Configuring an Etcd cluster is out of scope for this guide.

Using this Data Store you can store, read and delete data in the Etcd store.

|Option|Description|
|------|-----------|
|url|Where to find your etcd cluster, https is supported but not yet custom certs due to missing features in the etcdv3 gem.  Defaults to *http://127.0.0.1:2379*|
|user|Username to connect as, defaults to no username|
|password|Password to connect with, defaults to no password|

#### Shell Store

```yaml
$ds = {
  "type" => "shell",
  "command" => "/path/to/store.sh",
  "timeout" => 20,
  "cwd" => "/path/to",
  "environment" => {
    "EXAMPLE" => "VALUE"
  }
}
```

This type of store exists to make it easy for users to integrate existing data stores into the playbooks using just a shell command.

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

### Playbook level locks

Lets say you have a playbook that performs edits on a data base if more than one person ran this playbook at the same time you might destroy your data.  You want to be sure that the section in the playbook that touches the database is considered critical and can only run by one person on your network.

This is where locks come in, you can create arbitrary locks and wrap sections of the Playbook in locks:

```puppet
  $db_servers = choria::discover("mcollective",
    "discovery_method" => "choria",
    "classes"  => ["roles::primary_db"],
    "limit" => 1
  )

  $ds = {
    "type" => "consul",
    "timeout" => 120,
    "ttl" => 60
  }

  choria::lock("locks/db_critical", $ds) || {
    choria_task(
      "action" => "dbadmin.update_schema",
      "properties" => {"version" => "1.2.3"},
      "nodes" => $db_server
    )
  }
```

Any contents in the block will only execute if the lock could be held.  It does not matter what is in the block, only the name of the block.  So if you have many different kinds of DB related things you wish to run exclusively you can reuse the same lock name for them all.

There will be timeout for how long at most it will wait to get a lock, set using the *timeout* option when creating the data store.

When it makes sense the data store will create the lock in a way that should the playbook die, machine die or other unexpected thing the lock will timeout after a period configurable using the *ttl* option when creating the data store.
