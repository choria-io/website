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
   - [ ] It should support *macros* that ship with mcollective agent plugins and bring reusable self container utility logic
   - [ ] It should support versioned playbook schemas so we can ensure an incoming playbook is usable by the system ([#87](https://github.com/ripienaar/mcollective-choria/issues/87))
   - [ ] It should have JSON schemas for the full playbook schema - partially completed
   - [ ] It should validate incoming playbooks using the JSON schema
   - [ ] It should have an optional webservice where playbooks are stored and later ran under
   - [ ] It should be compatible with MCollective AAA, a webservice should be able to run commands as a certain user
   - [ ] It should produce reports that can be stored in JSON or in a webservice for later retrieval ([#88](https://github.com/ripienaar/mcollective-choria/issues/88))
   - [ ] It should support pluggable Node sources so users can extend it ([#90](https://github.com/ripienaar/mcollective-choria/issues/90))
   - [ ] It should support pluggable Task types so users can extend it ([#89](https://github.com/ripienaar/mcollective-choria/issues/89))
   - [ ] It should support arbitrarily named tasks lists and ability to run those from within a task list to facilitate reusing of logic ([#91](https://github.com/ripienaar/mcollective-choria/issues/91))

### Node Sets

   - [x] It should be able to do MCollective discovery
   - [x] It should be able to make arbitrary PQL queries
   - [x] It should be able to run shell scripts and get certnames from there
   - [x] It should be able to load node set groups from YAML
   - [x] It should be able to use terraform outputs
   - [ ] It should be able to get node groups from PE classifications
   - [ ] It should be able to get node groups from foreman classifications
   - [ ] It should be able to get all member nodes from a consul service group
   - [ ] It should be able to query EC2 and search by tag for nodes

### Tasks

   - [x] It should be able to make MCollective requests
   - [x] It should be able to make make assertions about MCollective request state
   - [x] It should be able to run local shell scripts
   - [x] It should be able to make GET and POST requests to arbitrary webhooks
   - [x] It should be able to send messages to Slack
   - [ ] It should be able to run terraform
   - [ ] It should be able to store data in consul
   - [ ] It should be able to store data in etcd
   - [ ] It should be able to initiate and wait for servers to be built by razor


Ticked features are implemented already in the POC and usable today.

A number of [issues have been opened on GitHub](https://github.com/ripienaar/mcollective-choria/issues?q=is%3Aissue+is%3Aopen+label%3Aplaybooks) to track work on this feature.

## Playbook Data Format

### JSON Schemas

The goal is to get JSON schemas for everyting setup, eventually this will be a, optional, webservice and I want swagger for everything.  A number of schemas in various states of completeness are in *schemas/playbook* on GitHub.

### Templating

Throughout - except in inputs - templating is supported to fill in values of inputs, metadata and node sets where required.

Generally should a template be the only thing in a string the data type of the referenced item is retained.  So *nodes: "{{{nodes.loadbalancers}}}"* will set the nodes to an array or nodes.  But if you did it mid some other part of a string it will use Ruby *to_s* to interprolate the result into your string.

#### input
Retrieves the value of a input.  Example *"{{{input.version}}}"* will rertieve the *version* input value, should this be an unknown input an error will be raised

#### nodes
Retrieves the list of nodes in a node set.  Example *"{{{nodes.loadbalancers}}}"* will rertieve the *loadbalancers* node set, should this be an unknown node set error will be raised

#### metadata
Retrieves any of the metadata items by name.  Example *"{{{metadata.version}}}"* will retrieve the *version* from metadata, unknown metadata items produce an error.


### Metadata

On the CLI the metadata is not that useful, down the line though these will live in a webservice and there you will want to search playbooks by author, tags, description and more.

```yaml
name: "test_playbook"
version: "1.1.2"
author: "R.I.Pienaar <rip@devco.net>"
description: "test description"
run_as: "deployer.bob"
loglevel: "info"

tags:
  - "test"
```

These are top level keys in the playbook structure.

#### name (required)
Unique name for this playbook matching

#### version (required)
Playbook version, only Semver is supported

#### author (required)
Author of the playbook, any name + email combo

#### description (required)
A textual description of this playbook - up to 256 characters

#### run_as
Reserved for future use, when run under a webservice the webservice will delegate the MCollective requests to this userid that you can use in AAA to control what the plabyook may do.  When run on the CLI it always runs as the user who is executing it.

#### loglevel (optional, defaults to info)
One of *debug*, *info*, *warn*

### MCollective agent version specification

There is no guarantee about versions of agents and clients in MCollective since it is completely losely coupled.  In an application like this you really do need to verify your expectations, playbooks support describing desired versions of agents that should exist on a specific *node set*.

```yaml
uses:
  rpcutil: "~1.0.0"
  puppet: "~1.11.0"
```

The names of of agents follow the usual MCollective naming rules.

This is simply pairs of agent and SemVer range specifications.  Later in node sets you'll see these used and the playbook runner will validate the network against these expectations.

### Inputs

Inputs are data items you can use to parameterise your playbooks, things like version of code to deploy or perhaps a identifier for a subset of nodes for Blue/Green deployments.

```yaml
inputs:
  cluster:
    description: "Cluster to deploy"
    type: "String"
    required: false
    validation: ":string"
    default: "alpha"

  two:
    description: "input 2 description"
    type: "String"
    validation: ":string"
```

Inputs are named according to generic MCollective item names.  Generally these map to what MCollective understands in Applicaiton plugin properties.

#### description (required)
Short description of the input

#### type (required)
Since the intention is that this must work on a JSON based webservice it will only support typical JSON types.  *String*, *Fixnum* or *Integer*, *:bool* and *:array*

#### validation
Validation is done using the MCollective validator framework, so anything you can put in there is valid here.  Be careful that these have to Symbol like as in the example above. Regular expressions are supported in strings like */foo/*.

#### required
Basic boolean to indicate if a value has to be provided or not

### default
Default value when not given

### Node Sets
Node sets are how you tell the playbook groups of named nodes to operate on.  These can be found via MCollective today but in future things like Consul, etcd, PE Console node groups, Terraform outputs, AWS API queries, shell scripts etc would be supported

```yaml
nodes:
  load_balancers:
    type: "mcollective"
    discovery_method: choria
    agents:
      - "puppet"
    at_least: 1
    when_empty: "No load balancers found with class haproxy"
    test: true
    uses:
      - "puppet"
      - "rpcutil"
```

Inputs are named according to generic MCollective item names.  Each type would have its own parameters, some would be genericly applied by the playbook framework to all node set types though.

#### type (required, meta)
A discovery type to use, today only *mcollective*

#### at_least (meta)
For any discovery type the playbook system would enforce that at least a certain amount of nodes are discovered by the discovery type

#### when_empty (meta)
When the *at_least* constraint is not satisfied this error / log will be produced

### limit (meta)
Limits the discovered nodes to a certain number, this is good to do canary deploys to pick a small subset of discovered nodes

#### discovery_method
For the MCollective type this picks the discovery method to use, valid values are whatever plugins are installed into your MCollective

#### agents, classes, facts, compound
For the MCollective type these are MCollective filters for use in discovery, same rules as usual for your MCollective

#### test
For the MCollective type when true this will do a *rpcutil#ping* against the discovered nodes to ensure they are managable and reachable, this is important if you use something like PuppetDB for discovery that is not aware of node reachability.

#### uses
This is a list of agents, these are pointers to the above mentioned MCollective version specifications.  The effect of this is that for the discovered nodes in the node set it will ensure that the listed agent match the desired SemVer specifications.  This is done by interrogating the nodes via MCollective just before running the playbook.

### Tasks
Tasks are the main place where you get things done, it is a list of steps to run through that just runs in sequence.  You cannot go backwards in the list and each step is run once.

Before, after and on success/fail certain hook task lists can be run, see the section on Hooks about that.

```yaml
tasks:
  - mcollective:
      description: "Disable Puppet on load balancers"
      nodes: "{{{nodes.load_balancers}}}"
      action: "puppet.disable"
      fail_ok: true
      tries: 2
      try_sleep: 10
      properties:
        :message: "hello world"
      post:
        - "summarize"
```

Here we have a *mcollective* type task with a number of properties:

#### description (required, meta)
A short description of what this task is doing

#### fail_ok (optional, meta)
When set to true should this step fail then that does not stop the playbook, its a soft failure that is logged but execution continues.  Default *false*

#### tries (optional, meta)
Should the task fail, it will be retried this many times.  The overall failure is determines at the end should the last try fail and max tries have been reached. Default *1*

#### try_sleep (optional, meta)
Should a retry need to be done this is how long it will sleep between attempts to run the task.  Default *10*

#### nodes
For the MCollective type this is a reference to a node set, here the templating is being used to tell MCollective to act on the discovered load balancers

#### action
For the MCollective type this is the agent and action to run, its a *.* seperated pair of agent and action.

#### properties
For the MCollective type this is a list of properties to pass to the action, unfortunately as MCollective is not JSON safe this for now might need to be Symbols as in the example

#### post
For the MCollective type this is a list of post action processing to do.  *summarize* means you want it to log the output from the aggregate plugins

#### silent
When true it will not log the raw RPC results to the log files

#### batch_size
Perform the operation in batches of size, it will execute the action on your nodes in groups

### Hook Task Lists
A number of specially named tasks lists are supported.  These support the exact same format of tasks as the main *tasks* list detailed above but they are only run under certain cases.

Here you would do remediation on failure, roll backs, notifications of Chat system, Open JIRA tickets etc.

#### pre_book
A task list that is run prior to the main task list, everything in this task list should pass else the main list wont be run.  Use this to for example notify about a deployment on Slack or perhaps to check if a maintenance window has been declared via a REST api call.

#### on_success
A task list that is run after the main task list completed without any error, should any tasks here fail it will mark the playbook as having failed

#### on_fail
A task list that ir run after the main task list completed with errors, as the playbook already failed the outcome of this list is effectively ignore, the playbook is always considered failed at this point

#### post_book
A task list that i run after the main task list and the on_success / on_fail hooks have completed.  A failure in this task list will mark a successful playbook as failed - but will not trigger the on_fail hook.
