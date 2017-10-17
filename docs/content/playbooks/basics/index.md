+++
title = "Basic Playbook"
weight = 400
+++

Lets consider a basic Playbook.  This will:

  * Pick nodes based on a *cluster* fact via PuppetDB based discovery
  * Test if they are reachable over mcollective
  * Verify their MCollective agent versions match our required versions as SemVer ranges
  * Disable Puppet on all the nodes
  * Wait for puppet to go idle
  * Stop the *httpd* service on all nodes with 2 retries
  * Run a app upgrade step on a custom agent to a specific version, 5 at a time, with retries, summarize the results
  * Start the *httpd* service on all nodes with 2 retries
  * Enable Puppet
  * Trigger a splay Puppet run
  * On success notify slack and a custom webhook
  * On failure notify slack and a custom webhook

Its quite limited for now but shows the basics

{{% notice tip %}}
The YAML format is a experiment, it just exposes the underlying data structures and as such is not very friendly. A new experimental [Puppet Plans based DSL](../plans/) exist which will be much more friendly.
{{% /notice %}}

```yaml
---
name: "app_upgrade"
version: "1.0.0"
author: "R.I.Pienaar <rip@devco.net>"
description: "Upgrades our acme application on a subset of nodes"
run_as: "choria=rip.mcollective"
loglevel: "info"

# which mcollective agents should exist and their versions
uses:
  rpcutil: "~1.0.0"
  puppet: "~1.11.0"
  acme: "~1.0.0"
  service: "~3.1.0"

# variables supplied on the CLI
inputs:
  cluster:
    description: "Cluster to deploy"
    type: "String"
    required: true
    validation: ":string"

  target_version:
    description: "Version of Acme to deploy"
    type: "String"
    require: true
    validation: ":string"

# Node sets discovered using the choria (PuppetDB) method on mcollective.
# Since PuppetDB is cached info, this will rpcutil ping them over
# mcollective to ensure they are alive
nodes:
  servers:
    type: mcollective
    discovery_method: choria
    when_empty: "No servers running Acme in cluster {{{ input.cluster }}} could be found"
    at_least: 1
    test: true
    facts:
      - "cluster={{{ input.cluster }}}"
    agents:
      - "acme"
    uses:
      - "rpcutil"
      - "puppet"
      - "acme"
      - "service"

# Main process to update our acme app
tasks:
  - mcollective:
      description: "Disable Puppet on Acme servers"
      nodes: "{{{ nodes.servers }}}"
      action: "puppet.disable"
      properties:
        :message: "Disabled during Acme update to version {{{ inputs.target_version }}}"

  - mcollective:
      description: "Wait for Puppet to go idle"
      action: "puppet.status"
      nodes: "{{{ nodes.servers }}}"
      assert: "idling=true"
      tries: 20
      try_sleep: 30

  - mcollective:
      description: "Disable httpd on Acme servers"
      nodes: "{{{ nodes.servers }}}"
      action: "service.stop"
      tries: 2
      try_sleep: 10
      properties:
        :service: "httpd"

  - mcollective:
      description: "Upgrade the Acme app to version {{{ inputs.target_version }}}"
      nodes: "{{{ nodes.servers }}}"
      action: "acme.upgrade"
      batch_size: 5
      tries: 2
      try_sleep: 20
      properties:
        :version: "{{{ inputs.target_version }}}"
      post:
        - "summarize"

  - mcollective:
      description: "Enable httpd on Acme servers"
      nodes: "{{{ nodes.servers }}}"
      action: "service.start"
      tries: 2
      try_sleep: 10
      properties:
        :service: "httpd"

  - mcollective:
      description: "Enable Puppet on Acme servers"
      nodes: "{{{ nodes.servers }}}"
      action: "puppet.enable"

  - mcollective:
      description: "Trigger a Puppet run on Acme servers"
      nodes: "{{{ nodes.servers }}}"
      action: "puppet.runonce"
      properties:
        :splay: true

# hooks for pre book, post book and fail/success handling exist,
# they are just task lists and can have many tasks
hooks:
  on_success:
    - slack:
        description: "Notify slack on success"
        token: "YOUR_API_TOKEN"
        channel: "#ops"
        text: "Acme was upgraded on cluster {{{ inputs.cluster }}} to version {{{ inputs.target_version }}}"

    - webhook:
        uri: https://hooks.example.net/webhook
        method: POST
        headers:
          "X-Acme-Token": "TOKEN"
        data:
          "message": "Deployed Acme release {{{ inputs.target_version }}}"
          "nodes": "{{{ nodes.servers }}}"

  on_fail:
    - slack:
        description: "Notify slack on failure"
        token: "YOUR_API_TOKEN"
        channel: "#ops"
        text: "Acme upgrade on cluster {{{ inputs.cluster }}} to version {{{ inputs.target_version }}} failed to complete"

    - webhook:
        uri: https://hooks.example.net/webhook
        method: POST
        headers:
          "X-Acme-Token": "TOKEN"
        data:
          "message": "Acme deployment to release {{{ inputs.target_version }}} failed"
          "nodes": "{{{ nodes.servers }}}"
```

You can run this playbook through the CLI, but lets look at help first, you can see our inputs are provided via *--cluster* and *--target_version*:

```bash
$ mco playbook playbook.yaml --help

Choria Playbook Runner

Usage:   mco playbook [OPTIONS] <ACTION> <PLAYBOOK>

  The ACTION can be one of the following:

    show      - preview the playbook
    run       - run the playbook as your local user

  The PLAYBOOK is a YAML file describing the tasks

  Passing --help as well as a PLAYBOOK argument will show
  flags and help related to the specific playbook.

  Any inputs to the playbook should be given on the CLI.

Application Options
        --cluster CLUSTER            Cluster to deploy (String)
        --target_version VERSION     Version of Acme to deploy (String)
        --loglevel LEVEL             Override the loglevel set in the playbook (debug, info, warn, error, fatal)
    -c, --config FILE                Load configuration from file rather than default
    -v, --verbose                    Be verbose
    -h, --help                       Display this screen

The Marionette Collective 2.9.1
```

And you can run the playbook using `mco playbook run playbook.yaml --cluster alpha --target_version 0.10`
