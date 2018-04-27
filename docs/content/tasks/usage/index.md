+++
title = "Usage"
weight = 200
+++

The majority of your interaction with this feature will be via the *mco tasks* cli.  It has extensive *--help* output which will not be reproduced here, sample commands will be shown here but please refer to the help for full details.

## Execution Flow

With the default Puppet Tasks runner from Puppet Inc the tasks are copied from your shell to remote nodes and ran there.  The flow in Choria is significantly different so I will show the steps that are followed when you execute a Task

  * The metadata is fetched from your Puppet Server and the CLI is created
  * Inputs are validated against the task signature
  * The task files specification are sent to all your nodes and they will attempt to download the task files
  * Request that nodes should run the Task - once it's confirmed that the entire node set has the exact same task files and commands downloaded and cached
  * Wait for up to 60 seconds for the Task to complete everywhere

At this point the task is either done or still executing - either way - you have a summary output that includes the task ID.

You can later retrieve task results, statusses and metadata from the nodes using the ID.

## Deploying a Task

Tasks are delivered using Puppet modules much like anything else in the Puppet world. Uniquely to Tasks you only have to put the files on your Puppet Server module paths, you do not need to include any classes etc.

{{% notice tip %}}
At present Choria will only consult your _production_ environment for tasks.
{{% /notice %}}

You can therefore use _puppet module_, _r10k_ or _librarian puppet_ to place your modules in the _production_ environment and that should be enough for them to be used by Choria

## Discovering Tasks

A basic list of tasks can be fetched from the CLI:

<pre><code class="nohighlight">
$ mco tasks
Known tasks in the production environment

  exec                                 gcompute::instance                   gcompute::reset
  gcompute::snapshot                   gcontainer::resize                   gpubsub::publish
  gsql::clone                          gsql::passwd                         gstorage::upload
  mcollective_agent_bolt_tasks::ping   puppet_conf

Use mco task <TASK> to see task help
Pass option --detail to see task descriptions
</code></pre>

Passing _--detail_ will request their descriptions, but please note that due to short comings in the Puppet API this is quite slow as a REST request has to be made for every task.

You can view details about one specific task:

<pre><code class="nohighlight">
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
</code></pre>

## Running a Task

Tasks are always run dissociated from the MCollectived, you can therefore use them to restart MCollective, do system upgrades and other long running tasks.

Lets look at the task input first

<pre><code class="nohighlight">
$ mco tasks run puppet_conf
The action option is mandatory
The setting option is mandatory

Please run with --help for detailed help
</code></pre>

<pre><code class="nohighlight">
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
</code></pre>

I've removed the bulk of the Help output here, the thing to note are the parameters like _--action, these are inputs defined by the task exposed on the CLI.

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

<pre><code class="nohighlight">
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
</code></pre>

Here as is typical for MCollective it will only show failures, you can later retrieve the output the command produced:

<pre><code class="nohighlight">
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
</code></pre>

Overview metadata about a task can be retrieved:

<pre><code class="nohighlight">
$ mco tasks status 45250c07824f5922be68468d08f6b76c --metadata -I node1.example.net
Requesting task metadata for request 45250c07824f5922be68468d08f6b76c

 * [ ============================================================> ] 1 / 1

  node1.example.net
    puppet_conf by choria=rip.mcollective at 2018-03-19 13:50:51
    completed: yes runtime: 1.39 stdout: yes stderr: no


Finished processing 1 / 1 hosts in 171.36 ms
</code></pre>

And finally you can retrieve all the output and statusses as JSON:

<pre><code class="nohighlight">
$ mco tasks status 45250c07824f5922be68468d08f6b76c --json -I node1.example.net|jq .
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
</code></pre>

## Discovery Integration

In the examples above it was kind of annoying to keep typing the hostname via _-I_, and it would not work really if you had many nodes or wanted to address only nodes that completed a task (or not).

Choria provides a discovery data source that expose a lot of data about a task to the MCollective discovery system.

Lets see what data the plugin provides and then we'll use what we saw to run a command on all nodes where a previous one failed:

<pre><code class="nohighlight">
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

</code></pre>

So there's a ton of information about past tasks, lets assume we ran a Task to upgrade some software and where the upgrade failed we have to do some remediation.

The original task that failed has id _ae561842dc7d5a9dae94f766dfb3d4c8_:

<pre><code class="nohighlight">
$ mco tasks run acme::recover -S "bolt_task('ae561842dc7d5a9dae94f766dfb3d4c8').exitcode > 0"
</code></pre>

This will query your network for all nodes where the task had a exitcode of greater than 0 and then it will run this _acme::recover_ task on them.
