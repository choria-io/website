+++
title = "Tasks"
weight = 430
toc = true
+++

Tasks are the main place you put things you want to happen.  Tasks goes into Task Lists and there are a few.

|List|Description|
|----|-----------|
|tasks|The main task list where you would your normal work|
|pre_book|A task list that runs before the main task list, a failure here prevents the list from running|
|on_success|A task list that runs when the main task list was succesfull, a failure here marks the book as failed|
|on_fail|A task list that runs when the main task list had a failure, passing here does not mark the playbook passed|
|post_book|A task list that runs after all the others, a failure here marks the book as failed|

All the lists work the same, so I will only show the main tasks here, see the basic example for one with other lists added.

## Common Task options

There are some items that can be applied to any type of task:

|Option|Description|Sample|
|------|-----------|------|
|tries |On failure, try this many times|*tries: 10*|
|try_sleep|Sleep this long between tries|*try_sleep: 60*|
|fail_ok|A failure will be logged but not fail the playbook|*fail_ok: true*|

## MCollective task

```yaml
tasks:
  mcollective:                                               # the type of task
    description: "Sample Task"
    nodes: "{{{ nodes.servers }}}"                           # a node set
    action: "puppet.disable"                                 # agent and action
    batch_size: 10                                           # in groups of 10 nodes
    silent: true                                             # dont log every result
    post:
      - summarize                                            # log aggregate summaries
    properties:                                              # input properties
      :message: "Disabled by playbooks"
```

{{% notice tip %}}
The *:message* here is a Symbol that is because today MCollective requires this, in future this will become JSO safe, for now this is required and dependant on the specific agent implementation.
{{% /notice %}}

|Option|Description|
|------|-----------|
|nodes|A node set to use, can be a template as here or just a hard coded array of nodes|
|action|A *agent* and *action* specified as one|
|batch_size|Optional setting to address the node set in smaller groups|
|silent|By default each result is logged, this disables that|
|post|Logs aggregate summaries if *summarize* - the only current valid entry - is given|
|properties|Any properties the action needs to function|

## MCollective Assert task

{{% notice tip %}}
This feature is included since *0.0.13*
{{% /notice %}}

This task is very similar to the MCollective one, it even uses it under the covers, but it's goal is to be used in cases where you want to wait for a certain condition to become true or just assert that it's true now.

The typical case is where you want to wait for a set of nodes to stop all Puppet runs before you make manual changes, this can be achieved using this task and the *puppet.status* action.

```yaml
  - mcollective_assert:
      description: Wait for Puppet to go idle
      action: puppet.status
      nodes: "{{{ nodes.dev }}}"
      expression:
        - :idling
        - "=="
        - true
      pre_sleep: 0
      tries: 10
      try_sleep: 30
```

Here we get the result from *puppet.status* and assert that all the nodes have their *:idling* result *true*.  Combining this with the *tries* and *try_sleep* gives us the wait behaviour vs just a one shot check.

|Option|Description|
|------|-----------|
|nodes|A node set to use, can be a template as here or just a hard coded array of nodes|
|action|A *agent* and *action* specified as one|
|properties|Any properties the action needs to function|
|pre_sleep|Sleep this long before doing the first check, useful if you did a task and have to wait for it to get going|
|expression|The test to perform, it's a array of 3 entries - the key to check, the expression and the value to check against|

Valid expressions are *<*, *>*, *>=*, *<=*, *==*, *=~*, *in*.  *=~* is for Regular Expressions and *in* checks if the left item is in the array on the right.

## Shell task

```yaml
tasks:
  - shell:
      description: Shell test
      command: "/home/rip/test.sh"
      cwd: "/tmp"
      nodes: "{{{ nodes.test_servers }}}"
      environment:
        "HELLO": "WORLD"
```

{{% notice warning %}}
If you pass any inputs into your script arguments be sure to set *validation: ":shellsafe"* on the input to ensure you are safe from shell injection attcks
{{% /notice %}}

|Option|Description|
|------|-----------|
|command|The command to run, can be a full command with options specified and all|
|cwd|The directory will be the working directory for the command when run|
|nodes|A list of nodes, passed as *--nodes node1,node2* to your script|
|environment|A hash of any environment variables you wish to set|

## Webhook task

{{% notice tip %}}
This feature is included since *0.0.13*
{{% /notice %}}

This allows arbitrary webhooks to be constructed and called via either GET or POST.

For a GET method the data gets URL encoded as request parts, for POST the data gets JSON encoded and sent.

```yaml
  - webhook:
      uri: https://hooks.example.net/webhook
      method: POST
      headers:
        "X-Token": "ho ho ho"
      data:
        "message": "Deployed Acme release {{{ inputs.version }}}"
        "nodes": "{{{ nodes.web_servers }}}"
```

The request will include a header *X-Choria-Request-ID* with a unique UUID for every request.

|Option|Description|
|------|-----------|
|uri|The address to send the request to, supports *http* or *https*|
|method|Either *GET* or *POST*|
|headers|Hash of headers to send.  It will send a User Agent and Content Type on it's own but you can override those too here|
|data|Hash of data to send either as request arguments for GET or JSON encoded data for POST|

## Slack task

Sends a message to a Slack channel, tested using the basic *bot user account* style bot.

```yaml
tasks:
  - slack:
      description: Slack test
      token: "xoxb-YOUR_TOKEN"
      channel: "#general"
      username: "Ops Bot"
      text: "Deployed Acme verion {{{ inputs.version }}} to cluster {{{ inputs.cluster }}}"
```

This posts a message to Slack with your text using the *chat.postMessage* API, above is all the possible options.
