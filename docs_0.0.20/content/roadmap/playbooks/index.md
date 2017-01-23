+++
title = "Playbook Design Specification"
weight = 410
toc = true
+++

## Introduction

See the [playbooks section](/docs/playbooks/) for intro material

## Features

This is a rough feature set, individual pieces are explored in more detail below:

### General

   - [x] It should support metadata like author, version, tags etc suitable for searching and cataloging on a web interface
   - [x] It should support many groups of named *node sets*, node set sources can be MCollective or many other sources
   - [x] It should support many named *inputs* usable from the CLI
   - [x] It should support declaring desired versions of MCollective agents and support verifying that nodes match expectation
   - [x] It should support tasks with common behaviours like retries and soft/hard failures etc
   - [x] It should support many types of task like MCollective RPC, Webhooks, Slack, Ticket systems etc
   - [x] It should support hook tasks to run on success, failure, between other tasks etc
   - [x] It should have a flexible playbook language that is pure data and support templating to fill in input data and reference node sets
   - [x] It should be standalone runable from the CLI
   - [x] It should be non interactive runs with rich reporting while being paramaterised
   - [x] It should have a custom logger format that shows the context as it moves through the playbook flow
   - [ ] It should have support for distributed key/value stores used in locks, inputs, node sets and tasks
   - [ ] It should support *macros* that ship with mcollective agent plugins and bring reusable self container utility logic
   - [ ] It should support versioned playbook schemas so we can ensure an incoming playbook is usable by the system ([#87](https://github.com/ripienaar/mcollective-choria/issues/87))
   - [ ] It should have JSON schemas for the full playbook schema - partially completed
   - [ ] It should validate incoming playbooks using the JSON schema
   - [ ] It should have an optional webservice where playbooks are stored and later ran under
   - [ ] It should be compatible with MCollective AAA, a webservice should be able to run commands as a certain user
   - [x] It should produce reports that can be stored in JSON or in a webservice for later retrieval ([#88](https://github.com/ripienaar/mcollective-choria/issues/88))
   - [ ] It should support pluggable Node sources so users can extend it ([#90](https://github.com/ripienaar/mcollective-choria/issues/90))
   - [ ] It should support pluggable Task types so users can extend it ([#89](https://github.com/ripienaar/mcollective-choria/issues/89))
   - [ ] It should support arbitrarily named tasks lists and ability to run those from within a task list to facilitate reusing of logic ([#91](https://github.com/ripienaar/mcollective-choria/issues/91))

### Inputs

   - [x] Defined inputs should be sourced from external systems when not supplied

### Node Sets

   - [x] It should be able to do MCollective discovery
   - [x] It should be able to make arbitrary PQL queries
   - [x] It should be able to run shell scripts and get certnames from there
   - [x] It should be able to load node set groups from YAML
   - [x] It should be able to use terraform outputs
   - [ ] It should be able to get node groups from PE classifications
   - [ ] It should be able to get node groups from foreman classifications
   - [ ] It should be able to query EC2 and search by tag for nodes
   - [ ] It should support data sources

### Tasks

   - [x] It should be able to make MCollective requests
   - [x] It should be able to make make assertions about MCollective request state
   - [x] It should be able to run local shell scripts
   - [x] It should be able to make GET and POST requests to arbitrary webhooks
   - [x] It should be able to send messages to Slack
   - [x] It should support data stores
   - [ ] It should be able to run terraform
   - [ ] It should be able to initiate and wait for servers to be built by razor

### Data Sources

   - [x] Should support dynamic inputs
   - [x] Should be able to write and delete from data stores in tasks
   - [x] Should have a in-memory data store
   - [ ] Should have a consul based data store
   - [ ] Should have a etcd based data store

Ticked features are implemented and usable today.

A number of [issues have been opened on GitHub](https://github.com/ripienaar/mcollective-choria/issues?q=is%3Aissue+is%3Aopen+label%3Aplaybooks) to track work on this feature.

## Thoughts

### Inputs

Today you have to specify inputs on the CLI but I think inputs would in real life be best sourced from elsewhere.

If you created a bunch of inputs and then defined an input source of say *etcd* any inputs not provided should be dynamically fetched from the input sources.

These inputs unlike CLI ones should be fetched every time a playbook reference them rather than just at the start.  This combined with tasks to set values in *etcd* and others could become quite powerful.

I think this has merit but won't rush into adding this feature till there is real use cases.

### State

I think with what we have quite a lot can be done but it's inevitable some state management will be needed.  If you act on something using a task it would be useful to always have the task outcomes available, imagine on failure you want to add some additional error message - the message from the last task would be handy.

I am not sure what more complex scenarios will exist but for sure the last task status should be surfaced via template variables, perhaps tasks could have names and then any task outcome could be referenced?

I suspect the task outcome format needs to be further formalised so that a state store can exist that's more full featured. One where once configured state will automatically be created in a k=v store and referenced in later tasks.

Again as with input sources I'd wait for real world use cases to exist.

### Deeper Consul and etcd integration

A deeper consul integration would make sense it could allow for:

  * Locks based on entire playbook to ensure it can run one a time only cross teams
  * Task level locks for same as above but more granular
  * Lock obtained as a task and kept till either later released or playbook exits
  * Already mentioned Inputs and Outputs
  * Service memberships
  * Events

A playbook might define a single named backend for this and whenever a lock/read/write etc is specified it would use this same backend definition

Some ideas for how this might look:

```yaml
---
name: app_upgrade
version: 1.0.0

data_stores:
  dc_store:
    type: consul
    ttl: 300            # session ttl, it will start a thread in the background keeping the session alive
    dc: foo             # else same dc as agent is in

# playbook level lock
lock:
  name: "dc_store/special_name" # else default is chosen
  timeout: 120                  # how long to wait at most for the lock to be gained

inputs:
  cluster:
    description: "Cluster to deploy"
    type: "String"
    required: true
    data: "dc_store/web_app_cluster_name" # key fetched from consul when not specifically given


nodes:
  foo_service:
    type: data_store
    service: "dc_store/foo_service"            # fetches the members from a service, not really usable in mco
                                               # but other tasks might enjoy them

tasks:
  - data:                                      # simple save/delete of any key nothing special
      action: "write"
      key: "dc_store/web_app_cluster_name"
      value: "alpha"

  - shell:
      description: "Upgrade web app"
      command: "/some/command --cluster {{{ inputs.cluster }}}" # if cluster is not specifically given, fetches dc_store/web_app_cluster_name
      lock:
        name: "dc_store/{{{ metadata.name }}}_upgrade_lock" # special name else default is chosen
        timeout: 120

  - data:
      action: "delete"
      key: "dc_store/web_app_cluster_name"
      options:
        recurse: true                         # type specific options should be possible like this consul specific one
```

So I guess the public API for these should be:

|Method|Description|
|------|-----------|
|lock  |Obtains a named lock for some TTL, in a background thread keeps the lock|
|release|Finds the background thread, kills it and releases the lock immediately|
|read|Reads a specific key from the kv store|
|write|Writes a key to the kv store|
|delete|Deletes a key from the kv store|
|service_members|Gets the members for a named service|

From there the tasks class can do task level locks, playbook class can do playbook level locks.  Inputs class can
manage dynamic lookups and will have to translate from the kv store value - a string - into the data type required
just like the CLI input on app does

Data task is a normal task that calls into this implementation
