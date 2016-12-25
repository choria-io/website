+++
title = "Tasks"
weight = 430
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
