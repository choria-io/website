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

### Inputs

   - [ ] Defined inputs should be sourced from external systems when not supplied
   - [ ] Query etcd for inputs
   - [ ] Query consul for inputs

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
