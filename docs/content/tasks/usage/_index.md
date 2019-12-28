+++
title = "Usage"
weight = 200
+++

## CLI

The CLI tools have extensive *--help* output which will not be reproduced here, sample commands will be shown here but please refer to the help for full details.

### Execution Flow

With the default Puppet Tasks runner from Puppet Inc the tasks are copied from your shell to remote nodes and run there.  The flow in Choria is significantly different so I will show the steps that are followed when you execute a Task

  * The metadata is fetched from your Puppet Server and the CLI is created
  * Inputs are validated against the task signature
  * The task files specification are sent to all your nodes and they will attempt to download the task files
  * Request that nodes should run the Task - once it's confirmed that the entire node set has the exact same task files and commands downloaded and cached
  * Wait for up to 60 seconds for the Task to complete everywhere

At this point the task is either done or still executing - either way - you have a summary output that includes the task ID.

You can later retrieve task results, statuses and metadata from the nodes using the ID.

### Deploying a Task

Tasks are delivered using Puppet modules much like anything else in the Puppet world. Uniquely to Tasks you only have to put the files on your Puppet Server module paths, you do not need to include any classes etc.

{{% notice tip %}}
At present Choria will only consult your _production_ environment for tasks.
{{% /notice %}}

You can therefore use _puppet module_, _r10k_ or _librarian puppet_ to place your modules in the _production_ environment and that should be enough for them to be used by Choria

### Discovering Tasks

A basic list of tasks can be fetched from the CLI:

```nohighlight
$ mco tasks
Known tasks in the production environment

  exec                                 gcompute::instance                   gcompute::reset
  gcompute::snapshot                   gcontainer::resize                   gpubsub::publish
  gsql::clone                          gsql::passwd                         gstorage::upload
  mcollective_agent_bolt_tasks::ping   puppet_conf

Use mco task <TASK> to see task help
Pass option --detail to see task descriptions
```

Passing _--detail_ will request their descriptions, but please note that due to shortcomings in the Puppet API this is quite slow as a REST request has to be made for every task.

You can view details about one specific task:

```nohighlight
$ mco tasks puppet_conf
Retrieving task metadata for task puppet_conf from the Puppet Server

puppet_conf - Inspect puppet agent configuration settings

Task Parameters:
  action                         The operation (get, set) to perform on the configuration setting (Enum[get, set])
  section                        The section of the config file. Defaults to main (Optional[String[1]])
  setting                        The name of the config entry to set/get (String[1])
  value                          The value you are setting. Only required for set (Optional[String[1]])

Task Files:
  init.rb                        1231 bytes

Use 'mco tasks run puppet_conf' to run this task
```

### Running a Task

Tasks are always run dissociated from the *mcollectived*, you can therefore use them to restart MCollective, do system upgrades and other long running tasks.

Lets look at the task input first

```nohighlight
$ mco tasks run puppet_conf
The action option is mandatory
The setting option is mandatory

Please run with --help for detailed help
```

```nohighlight
$ mco tasks run puppet_conf --help

Puppet Task Orchestrator

Usage:
    mco tasks run <TASK NAME> [OPTIONS]

 Runs a task in the background and wait up to 50 seconds for it to complete.

 ....

Application Options
        --action ACTION              The operation (get, set) to perform on the configuration setting (Enum[get, set])
        --section SECTION            The section of the config file. Defaults to main (Optional[String[1]])
        --setting SETTING            The name of the config entry to set/get (String[1])
        --value VALUE                The value you are setting. Only required for set (Optional[String[1]])
        --summary                    Only show a overall summary of the task
        --background                 Do not wait for the task to complete
        --input INPUT                JSON input to pass to the task
    -c, --config FILE                Load configuration from file rather than default
    -v, --verbose                    Be verbose
    -h, --help                       Display this screen
```

I've removed the bulk of the Help output here, the thing to note are the parameters like _--action_, these are inputs defined by the task exposed on the CLI.

In most cases Choria will attempt to convert a string like _1_ into an Integer when that is what the task need so for most tasks you can pass the options as normal.  You can though also use a YAML or JSON file as input, for example:

```json
{
  "action":"set",
  "section":"user",
  "setting":"modulepath",
  "value":"/tmp/modules"
}
```

You could save this file to _input.json_ and run *mco tasks run puppet_conf --input @input.json* and it will use these settings, here we'll show supplying them on the CLI though:

```nohighlight
$ mco tasks run puppet_conf --action set --section user --setting modulepath --value /tmp/modules -I node1.example.net
Retrieving task metadata for task puppet_conf from the Puppet Server
Attempting to download and run task puppet_conf on 1 nodes

Downloading and verifying 1 file(s) from the Puppet Server to all nodes: ✓  1 / 1
Running task puppet_conf and waiting up to 60 seconds for it to complete



Summary for task 45250c07824f5922be68468d08f6b76c

                       Task Name: puppet_conf
                          Caller: choria=rip.mcollective
                       Completed: 1
                         Running: 0

                      Successful: 1
                          Failed: 0

                Average Run Time: 1.39s
```

Here as is typical for MCollective it will only show failures, you can later retrieve the output the command produced:

```nohighlight
$ mco tasks status 45250c07824f5922be68468d08f6b76c -v -I node1.example.net
Discovering hosts using the choria method .... 1

node1.example.net
   {"status":"/tmp/modules","setting":"modulepath","section":"user"}



Summary for task 45250c07824f5922be68468d08f6b76c

                       Task Name: puppet_conf
                          Caller: choria=rip.mcollective
                       Completed: 1
                         Running: 0

                      Successful: 1
                          Failed: 0

                Average Run Time: 1.39s
```

Overview metadata about a task can be retrieved:

```nohighlight
$ mco tasks status 45250c07824f5922be68468d08f6b76c --metadata -I node1.example.net
Requesting task metadata for request 45250c07824f5922be68468d08f6b76c

 * [ ============================================================> ] 1 / 1

  node1.example.net
    puppet_conf by choria=rip.mcollective at 2018-03-19 13:50:51
    completed: yes runtime: 1.39 stdout: yes stderr: no


Finished processing 1 / 1 hosts in 171.36 ms
```

And finally you can retrieve all the output and statuses as JSON:

```nohighlight
$ mco tasks status 45250c07824f5922be68468d08f6b76c --json -I node1.example.net | jq .
{
  "items": [
    {
      "host": "node1.example.net",
      "task": "puppet_conf",
      "callerid": "choria=rip.mcollective",
      "exitcode": 0,
      "stdout": {
        "status": "/tmp/modules",
        "setting": "modulepath",
        "section": "user"
      },
      "stderr": "",
      "completed": true,
      "runtime": 1.386015831,
      "start_time": 1521467451
    }
  ],
  "stats": {
    "names": [
      "puppet_conf"
    ],
    "callers": [
      "choria=rip.mcollective"
    ],
    "completed": 1,
    "running": 0,
    "task_not_known": 0,
    "wrapper_failure": 0,
    "success": 1,
    "failed": 0,
    "average_runtime": 1.386015831,
    "noresponses": []
  },
  "summary": {
    "nodes": 1,
    "taskid": "45250c07824f5922be68468d08f6b76c",
    "completed": true,
    "success": true
  }
}
```

### Discovery Integration

In the examples above it was kind of annoying to keep typing the hostname via _-I_, and it would not work really if you had many nodes or wanted to address only nodes that completed a task (or not).

Choria provides a discovery data source that exposes a lot of data about a task to the MCollective discovery system.

Let's see what data the plugin provides and then we'll use what we saw to run a command on all nodes where a previous one failed:

```nohighlight
$ mco plugin doc data/bolt_task
bolt_task
=========

Information about past Bolt Task

      Author: R.I.Pienaar <rip@devco.net>
     Version: 0.6.0
     License: Apache-2.0
     Timeout: 1
   Home Page: https://choria.io

   This data plugin let you extract information about a previously
   run Bolt Task for use in discovery and elsewhere.

   To run a task on nodes where one previously failed:

      mco tasks run myapp::update -S "bolt_task('ae561842dc7d5a9dae94f766dfb3d4c8').exitcode > 0"

QUERY FUNCTION INPUT:

              Description: The Task ID to retrieve
                   Prompt: Task ID
                     Type: string
               Validation: ^[a-z,0-9]{32}$
                   Length: 32

QUERY FUNCTION OUTPUT:

           caller:
              Description: The user who invoked the task
               Display As: Invoked by

           completed:
              Description: Did the task complete running
               Display As: Completed

           exitcode:
              Description: The exitcode from the task
               Display As: Exit Code

           known:
              Description: If this is a known task on this node
               Display As: Known Task

           runtime:
              Description: How long the task took to run
               Display As: Runtime

           spool:
              Description: Where on disk the task status is stored
               Display As: Spool

           start_time:
              Description: When the task was started, seconds since 1970 in UTC time
               Display As: Start Time

           stderr:
              Description: The STDERR output from the task
               Display As: STDERR

           stdout:
              Description: The STDOUT output from the task
               Display As: STDOUT

           task:
              Description: The name of the task that was run
               Display As: Task

           wrapper_error:
              Description: Error output from the wrapper command
               Display As: Wrapper Error

           wrapper_pid:
              Description: The PID of the wrapper that runs the task
               Display As: Wrapper PID

           wrapper_spawned:
              Description: Did the wrapper start successfully
               Display As: Wrapper Spawned
```

So there's a ton of information about past tasks, lets assume we ran a Task to upgrade some software and where the upgrade failed we have to do some remediation.

The original task that failed has id _ae561842dc7d5a9dae94f766dfb3d4c8_:

```nohighlight
$ mco tasks run acme::recover -S "bolt_task('ae561842dc7d5a9dae94f766dfb3d4c8').exitcode > 0"
```

This will query your network for all nodes where the task had a exitcode of greater than 0 and then it will run this _acme::recover_ task on them.

## Playbooks

As of version 0.9.0 of the *choria/choria* module you can use Tasks from within Choria Playbooks.

### Running a Task

The integration is intended to be used by other Playbooks, let's look at using the [Package Task](https://forge.puppet.com/puppetlabs/package) to retrieve the version of the *puppet-agent* package:

```puppet
plan acme::agent_status () {
  $nodes = choria::discover(
    "agents" => ["bolt_tasks"]
  )

  $results = choria::run_playbook("choria::tasks::run",
    "nodes" => $nodes,
    "task" => "package",
    "silent" => true,
    "inputs" => {
      "name" => "puppet-agent",
      "action" => "status"
    }
  )

  $results.reduce({}) |$memo, $result| {
    $memo + {$result.host => $result["data"]["stdout"].parsejson}
  }
}
```

Here we use the *choria::tasks::run* playbook to perform the same basic Flow that the CLI does - get metadata, validate input, download task files, run task and wait up to 60 seconds.

{{% notice tip %}}
The examples shown were made using our [Vagrant Demo](https://github.com/choria-io/vagrant-demo) environment
{{% /notice %}}

We use the *Choria::Results* set to create a simplified result status, this might look like this:

```nohighlight
$ mco playbook run acme::agent_status
Notice: Scope(<module>/acme/plans/agent_status.pp, 2): Discovered 3 node(s) in node set task_nodes
Info: Scope(<module>/choria/plans/tasks/download_files.pp, 12): Downloading files for task 'package' onto 3 nodes
Notice: Scope(<module>/choria/plans/tasks/download_files.pp, 14): About to run task: mcollective task
Notice: Scope(<module>/choria/plans/tasks/download_files.pp, 14): Starting request for bolt_tasks#download against 3 nodes
Notice: Scope(<module>/choria/plans/tasks/download_files.pp, 14): Successful request de3266400e165b76aa165c1f29f1063c for bolt_tasks#download in 1.01s against 3 node(s)
Info: Scope(<module>/choria/plans/tasks/run.pp, 43): Running task 'package' on 3 nodes
Notice: Scope(<module>/choria/plans/tasks/run.pp, 45): About to run task: mcollective task
Notice: Scope(<module>/choria/plans/tasks/run.pp, 45): Starting request for bolt_tasks#run_and_wait against 3 nodes
Notice: Scope(<module>/choria/plans/tasks/run.pp, 45): Successful request ca87087b91f950b6a226dd52d33f7c42 for bolt_tasks#run_and_wait in 3.62s against 3 node(s)

Plan acme::agent_status ran in 5.93 seconds: OK

Result:

     {
       "choria1.choria": {
         "status": "up to date",
         "version": "5.5.1-1.el7"
       },
       "choria0.choria": {
         "status": "up to date",
         "version": "5.5.1-1.el7"
       },
       "puppet.choria": {
         "status": "up to date",
         "version": "5.5.1-1.el7"
       }
     }
```

The *choria::tasks::run* playbook supports options for batching requests and more, please review the comments in its code.

This is a normal task invocation, so you could for example use *mco tasks status ca87087b91f950b6a226dd52d33f7c42* to later look at this task from the CLI.

### Retrieving task Status

You can also retrieve the status for a specific task in a Playbook:

```puppet
  # ....

  $results = choria::run_playbook("choria::tasks::status",
    "nodes" => $nodes,
    "task_id" => "ca87087b91f950b6a226dd52d33f7c42",
  )

  # ....
```

### Waiting for a task to complete

And finally if you started a task across a set of nodes and this task might run for a very long time you can pause your Playbook while this is happening:

```puppet
  # ...
  choria::run_playbook("choria::tasks::wait",
    "nodes" => $nodes,
    "task_id" => "ca87087b91f950b6a226dd52d33f7c42",
  )

  # ...
```

Here the Playbook system will regularly fetch the status and wait for all nodes to complete.  By default it will check 90 times with a 20 second sleep - so for 30 minutes.  You can influence these parameters using *tries* and *sleep* arguments to the *choria::tasks::wait* playbook.
