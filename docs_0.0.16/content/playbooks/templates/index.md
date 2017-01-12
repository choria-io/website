+++
title = "Templates"
weight = 440
toc = true
+++

Today playbooks support a very limited set of template expansion, I really want to avoid people writing programs in their data so it's very limited now until we find some real use cases.

The basic format of a template is `{{{ something }}}` where the leading and trailing whitespace is optional

These templates are data type aware so when a string is entirely just a lookup of an array like a node list, then the data becomes an array:

```yaml
nodes: "{{{ nodes.webservers }}}"
```

will be the same as:

```yaml
nodes:
 - server1.example.net
 - server2.example.net
```

But by contrast this will be a string:

```yaml
message: "the list of servers: {{{ nodes.webservers }}}"
```

will be the same as:

```yaml
message: "the list of servers: server1.example.net, server2.example.net"
```

Likewise Hash data will be shown as JSON data, everything else will be unpredictable format `to_s`.

## Metadata
Metadata is the stuff you can find at the top of your playbooks:

```yaml
name: "app_upgrade"
version: "1.0.0"
author: "R.I.Pienaar <rip@devco.net>"
description: "Upgrades our acme application on a subset of nodes"
run_as: "choria=rip.mcollective"
loglevel: "info"
on_fail: "fail"
tags:
  - acme
```

Any of this stuff can be retrieved using `{{{ metadata.loglevel }}}`, trying to access a key other than these will yield an exception

## Inputs
Any input you define can be retrieved anywhere in a playbook - except metadata and other inputs.

```yaml
inputs:
  cluster:
    description: "Cluster to deploy"
    type: "String"
    required: true
    validation: ":shellsafe"
```

To access this item you can just do `{{{ input.cluster }}}` or `{{{ inputs.cluster }}}`

## Nodes
Node sets will be arrays, as shown above you can either stick them mid string for a comma joined list or keep the true array form.

Here the nodes will be an array:

```yaml
tasks:
  - mcollective:
      nodes: "{{{ nodes.servers }}}"
```

But here it will be a comma joined string:

```yaml
- webhook:
    uri: https://hooks.example.net/webhook
    method: POST
    headers:
      "X-Acme-Token": "TOKEN"
    data:
      "message": "Acme deployed to release {{{ inputs.version }}} on nodes {{{ nodes.servers }}}"
```

## Previous Task Status

{{% notice tip %}}
As of version *0.0.16*
{{% /notice %}}

You can reference items from the previous task via templates, only the one previous item is available
and if no previous task ran informative data will be returned.  This is designed to use in *on_fail*
tasks lists to notify Slack with context.

```yaml
hooks:
  on_fail:
    - slack:
        description: "Notify slack"
        token: "YOUR_API_TOKEN"
        channel: "#general"
        text: "Playbook `{{{ metadata.name }}}` task `{{{ previous_task.description }}}` against nodes `{{{ nodes.dev }}}` failed after `{{{ previous_task.runtime }}}` seconds with: ```{{{ previous_task.msg }}}```"
```

|Item|Description|
|----|-----------|
|previous_task.success|Boolean indicating if the task ran, false when not ran at all|
|previous_task.description|The task description|
|previous_task.msg or previous_task.message|The status message the previous task produced|
|previous_task.data|The raw data the task produced, will be shown using *inspect* from Ruby|
|previous_task.runtime|How long the task ran in seconds|

## Times
{{% notice tip %}}
As of version *0.0.14*
{{% /notice %}}

You might need the time to send off in webhooks or to slack etc, you can format the current time in either local or UTC time:

Here we create a RFC3339 format time UTC time stamp:

```yaml
message: "Acme deployed at {{{ utc_date(%FT%T) }}}"
```

And here a local time version of the same:

```yaml
message: "Acme deployed at {{{ date(%FT%T) }}}"
```

Formats are as per ruby `Time#strftime` format and you must supply a format.

## UUIDs
{{% notice tip %}}
As of version *0.0.14*
{{% /notice %}}

At present only UUIDs created using the `MCollective::SSL.uuid` method is supported, here we create a unique ID in a Webhook:

```yaml
- webhook:
    uri: https://hooks.example.net/webhook
    method: POST
    headers:
      "X-Acme-Request": "{{{ uuid }}}"
    data:
      "message": "Acme deployed to release {{{ inputs.version }}} on nodes {{{ nodes.servers }}}"
```
