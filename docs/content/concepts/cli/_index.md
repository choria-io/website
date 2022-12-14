+++
title = "CLI Interaction Model"
weight = 205
toc = true
+++

Choria is designed first and foremost for the CLI. You will mostly
interact with a single executable called *mco* which has a number of
sub-commands, arguments and flags.

Choria, being MCollective compatible, relies on the MCollective CLI toolkit
for its interactions, if you've previously used MCollective it should be
very familiar.

{{% notice tip %}}
These examples can be tried using our [Vagrant Demo](https://github.com/choria-io/vagrant-demo) environment
{{% /notice %}}

{{% notice info %}}
At present we are migrating most of our command line tooling from `mco` to `choria`, this document switches around a bit, we'll improve consistency in time. `mco` is used for those commands that is only implemented in the Ruby language.
{{% /notice %}}

## Basic Usage of the *mco* Command

A simple example of a *mco* command can be seen below:

```nohighlight
$ choria ping
choria1.choria                           time=30.49 ms
puppet.choria                            time=30.71 ms
choria0.choria                           time=31.47 ms

---- ping statistics ----
3 replies max: 31.47 min: 30.49 avg: 30.89
```

In this example the *ping* sub-command is referred to as an
*application*. Choria provides many applications, for a list of
them, type *mco help*. You can also create your own application to plug
into the framework. The *help* sub-command will show you something like
this:

```nohighlight
$ mco help
The Marionette Collective version 2.12.1

  choria          Choria Orchestrator Management and Configuration
  completion      Helper for shell completion systems
  describe_filter Display human readable interpretation of filters
  facts           Reports on usage for a specific fact
  federation      Choria Federation Broker Utilities
  filemgr         Generic File Manager Client
  find            Find hosts using the discovery system matching filter criteria
  help            Application list and help
  inventory       General reporting tool for nodes, collectives and subcollectives
  nettest         Network tests from a mcollective host
  package         Install, uninstall, update, purge and perform other actions to packages
  ping            Ping all nodes
  playbook        Choria Playbook Runner
  plugin          MCollective Plugin Application
  process         Distributed Process Management
  puppet          Schedule runs, enable, disable and interrogate the Puppet Agent
  rpc             Generic RPC agent client application
  service         Manages system services
  shell           Run shell commands
  tasks           Puppet Task Orchestrator
```

You can request help for a specific application using either *mco help
application* or *mco application ---help*. Shown below is part of the
help for the *rpc* application:

```nohighlight
$ choria rpc --help
usage: choria req [<flags>] <agent> <action> [<args>...]

Performs a RPC request against the Choria network

Replies are shown according to DDL rules or --display, replies can also be filtered using an expression language that will entirely remove them from the returned result.

  # remove all ok results, only errors are in the output and json
  --filter-replies 'IsOK()'

  # remove all puppet status replies where the machines are idling
  --filter-replies '!IsOK() || Data("idling")'

  # remove all replies where the last value of the array matches
  --filter-replies 'Data("array.@reverse.0")=="last_item"'

The Data() function here uses gjson Path Syntax.

Flags:
      --help                  Show context-sensitive help (also try --help-long and --help-man).
      --version               Show application version.
  -d, --debug                 Enable debug logging
      --config=FILE           Config file to use
  -F, --wf=WF ...             Match hosts with a certain fact
  -C, --wc=WC ...             Match hosts with a certain configuration management class
  -A, --wa=WA ...             Match hosts with a certain Choria agent
  -I, --wi=WI ...             Match hosts with a certain Choria identity
  -W, --with=FILTER ...       Combined classes and facts filter
      --limit=LIMIT           Limits request to a set of targets eg 10 or 10%
      --limit-seed=SEED       Seed value for deterministic random limits
      --batch=SIZE            Do requests in batches
      --batch-sleep=SECONDS   Sleep time between batches
  -T, --target=TARGET         Target a specific sub collective
      --workers=3             How many workers to start for receiving messages
      --nodes=NODES           List of nodes to interact with
      --np                    Disable the progress bar
  -j, --json                  Produce JSON output only
      --table                 Produce a Table output of successful responses
  -v, --verbose               Enable verbose output
      --display=DISPLAY       Display only a subset of results (ok, failed, all, none)
      --discovery-timeout=SECONDS
                              Timeout for doing discovery
      --dm=DM                 Sets a discovery method (broadcast, choria)
  -o, --output-file=FILENAME  Filename to write output to
      --filter-replies=EXPR   Filter replies using a expr filter

Args:
  <agent>   The agent to invoke
  <action>  The action to invoke
  [<args>]  Arguments to pass to the action in key=val format
```

The *help* first shows a basic overview of the command line syntax
followed by options specific to this command.  Following that you will
see some *Common Options* and *Host Filters* that generally apply to
most applications.

## Making Agent Requests

### Overview of a Request

The *rpc* application is the main application used to make requests to
your servers. It is capable of interacting with any standard Remote
Procedure Call (RPC) agent. Below is an example that shows an attempt to
restart a ssh server on several machines:

```nohighlight
$ choria rpc service restart service=sshd
Discovering hosts using the mc method for 2 second(s) .... 3

 * [ ============================================================> ] 3 / 3


puppet.choria                            Unknown Request Status
   Systemd restart for sshd failed!
journalctl log for sshd:
-- Logs begin at Thu 2018-05-03 10:20:09 UTC, end at Thu 2018-05-03 10:33:40 UTC. --
May 03 10:33:40 puppet.choria systemd[1]: Stopping OpenSSH server daemon...
May 03 10:33:40 puppet.choria systemd[1]: Starting OpenSSH server daemon...
May 03 10:33:40 puppet.choria systemd[1]: sshd.service: main process exited, code=exited, status=203/EXEC
May 03 10:33:40 puppet.choria systemd[1]: Failed to start OpenSSH server daemon.
May 03 10:33:40 puppet.choria systemd[1]: Unit sshd.service entered failed state.
May 03 10:33:40 puppet.choria systemd[1]: sshd.service failed.



Summary of Service Status:

   running = 2
   unknown = 1


Finished processing 3 / 3 hosts in 542.48 ms
```

The order of events in this process are:

 * Perform discovery against the network and discover 10 servers
 * Send the request and then show a progress bar of the replies
 * Show any results that were out of the ordinary
 * Show some statistics

Choria client applications aim to only provide the most relevant
information.  In this case, the application is not showing verbose
information about the nine *OK* results, since the most important issue
is the one *Failure*. Keep this in mind when viewing the results of
commands.

### Anatomy of a Request

Choria agents are broken up into actions and each action can take
input arguments.

```nohighlight
$ choria rpc service stop service=sshd
```

This shows the basic make-up of an RPC command. In this case we are:

 * using the *rpc* application - a generic application that can interact with any agent
 * directing our request to machines with the *service* agent
 * sending a request to the *stop* action of the service agent
 * supplying a value, *sshd*, to the *service* argument of the *stop* action

The same command has a longer form as well:

```nohighlight
$ choria rpc --agent service --action stop --argument service=sshd
```

These two commands are functionally identical.

### Discovering Available *Agents*

The above command showed you how to interact with the *service* agent,
but how can you find out that this agent even exists? On a correctly
installed Choria system you can use the *plugin* application to get
a list:

```nohighlight
$ choria plugin doc
Known Plugins:

Agents:

          aaa_signer: Request Signer for Choria AAA Service
          bolt_tasks: Downloads and runs Puppet Tasks
         choria_util: Choria Utilities
             package: Manage Operating System Packages
              puppet: Manages the Life Cycle of the Puppet Agent
             rpcutil: Utility actions that expose information about the state of the running Server
             service: Manages Operating System Services

Data Providers:

              choria: Data about a the running Choria instance
         config_item: Runtime value of a configuration items
       machine_state: Data about a Choria Autonomous Agent
               scout: Data about a specific Scout check


```

The first part of this list shows all the agents this computer is aware
of. In order to show up on this list, an agent must have a *DDL* file
and be installed locally.

To find out the *actions*, *inputs* and *outputs* for a specific agent
use the plugin application again:

```nohighlight
$ choria plugin doc agent/service
Service Agent version 4.0.1

Manages Operating System Services

Plugin Metadata:

    Author: R.I.Pienaar <rip@devco.net>
       URL: https://github.com/choria-plugins/service-agent
   License: Apache-2.0
   Timeout: 60

Available Action: restart, start, status, stop

Actions:

  start Action:

    Start a service
    
    Inputs:
    
      service:
        
        The service to start
        
        ║      Prompt: Service Name
        ║        Type: String
        ║    Optional: false
        ║  Validation: service_name
        ║  Max Length: 90
        ╙─
        
    
    Outputs:
    
      status:
        
        The status of the service after starting
        
        ║  Display As: Service Status
        ║        Type: Undefined
        ║     Default: unknown
        ╙─
....
```

This shows a truncated example of the auto-generated help for the
*service* agent. First shown is metadata such as version, author and
license. This is followed by the list of actions available, in this case
the *restart*, *start*, *status* and *stop* actions.

Further information is shown about each action. For example, you can see
that the *status* action requires an input called *service* which is a
string, has a maximum length of 30, etc. You can also see you will
receive one output called *status*

With this information, you can request the status for a specific
service:

```nohighlight
$ choria rpc service status service=sshd
Discovering hosts using the mc method for 2 second(s) .... 3

 * [ ============================================================> ] 3 / 3


choria1.choria
   Service Status: running

puppet.choria
   Service Status: stopped

choria0.choria
   Service Status: running


Summary of Service Status:

   running = 2
   stopped = 1


Finished processing 3 / 3 hosts in 21.35 ms
```

Unlike the previous example, in this case specific information is
returned on the success of the action. This is because this specific
action is meant to retrieve information and so Choria assumes you
would like to see complete, thorough data regardless of success or
failure.

Note that this output displays *Service Status* as shown in the *mco
plugin doc service* help page. Any time you need more information about
a display name, the doc for the associated agent will have a
*Description* section for every input and output.

## Selecting Request Targets Using *Filters*

Filters are how you use our discovery system, this is extensively documented in [../discovery](Discovery).

### Basic Filters

A key capability of Choria is fast discovery of network resources.
Discovery rules are written using *filters*.  For example:

```nohighlight
$ choria req service status service=httpd -W "environment=development customer=acme"
```

This shows a filter rule that limits the RPC request to being run on
machines that are both in the Puppet environment *development* and
belong to the Customer *acme*.

Filtering can be based on *facts*, the presence of a *Configuration
Management Class* on the node, the node's *Identity*, or installed
*Agents* on the node.

Here are a number of examples of this with short descriptions of each
filter:

```nohighlight
# all machines with the service agent
$ choria ping -A service
$ choria ping --with-agent service

# all machines with the apache class on them
$ choria ping -C apache
$ choria ping --with-class apache

# all machines with a class that match the regular expression
$ choria ping -C /service/

# all machines in the UK
$ choria ping -F country=uk
$ choria ping --with-fact country=uk

# all machines in either UK or USA
$ choria ping -F "country=/uk|us/"

# just the machines called dev1 or dev2
$ choria ping -I dev1 -I dev2

# all machines in the domain foo.com
$ choria ping -I /foo.com$/
```

As you can see, you can filter by Agent, Class and/or Fact, and you can
use regular expressions almost anywhere.  You can also combine filters
additively in a command so that all the criteria have to be matched.

Note: You can use a shortcut to combine Class and Fact filters:

```nohighlight
# all machines with classes matching /apache/ in the UK
$ choria ping -W "/apache/ location=uk"
```

## Complex Compound or Select Queries

{{% notice tip %}}
This is an experimental feature and subject to change
{{% /notice %}}

While the above examples are easy to enter, they are limited in that they can only combine filters additively. If you want to create searches with more complex boolean logic use the -S switch. For example:

```nohighlight
$ choria ping -S '((with("customer=acme") && with("environment=staging") || with("environment=development")) && with("/apache/")'
```

The above example shows a scenario where the development environment is usually labeled development but one customer has chosen to use staging. You want to find all machines in those customers' environments that match the class apache. This search would be impossible using the previously shown methods, but the above command uses -S to allow the use of boolean operators such as and and or so you can easily build the logic of the search.

A more complex example can be seen here:

```nohighlight
with('apache') and                              # class or agent 'apache'
  with('/t.sting/') and                         # class or agent regex match 't.sting'
  with('fnumber=1.2') and                       # fact fnumber with a float value equals 1.2
  fact('nested.string') matches('h.llo') and    # lookup a fact 'nested.string' and regex match it with 'h.llo'
  include(fact('sarray'), '1') and              # check if the 'sarray' fact - a array of strings - include a value '1'
  include(fact('iarray'), 1)                    # check if the 'iarray' fact - a array of ints - include a value 1
```

These expressions are built using [expr](https://github.com/antonmedv/expr/blob/master/docs/Language-Definition.md) and [GJSON path syntax](https://github.com/tidwall/gjson/blob/master/SYNTAX.md). 

Within the expressions we have defined some variables:

| Variable  | Description                             |
|-----------|-----------------------------------------|
| `agents`  | List of known agents                    |
| `classes` | List of classes this machine belongs to |
| `facts`   | Facts for the machine as raw JSON       |

And we made a few functions available:

| Function  | Description                                                                                                                  |
|-----------|------------------------------------------------------------------------------------------------------------------------------|
| `with`    | Equivalent of a `-W` filter - class and fact matches combined                                                                |
| `fact`    | Retrieves a fact from the nested fact data using [GJSON path syntax](https://github.com/tidwall/gjson/blob/master/SYNTAX.md) |
| `include` | Checks if an array includes a specific element                                                                               |

## Tabular format 

The *rpc* application defaults to a human friendly format, you can activate an 
alternative format that is a table of data:

```nohighlight
$ choria rpc package status package=puppet-agent -I choria0.choria --table
Discovering nodes using the choria method ....1

1 / 1    0s [====================================================================] 100%

+----------------+--------+--------------+-------+--------------+--------+----------+---------+---------+
| SENDER         | ARCH   | ENSURE       | EPOCH | NAME         | OUTPUT | PROVIDER | RELEASE | VERSION |
+----------------+--------+--------------+-------+--------------+--------+----------+---------+---------+
| choria0.choria | x86_64 | 6.17.0-1.el7 | 0     | puppet-agent |        | yum      | 1.el7   | 6.17.0  |
+----------------+--------+--------------+-------+--------------+--------+----------+---------+---------+
```

## Seeing the Raw Data

By default, the *rpc* application will try to show human-readable data.
To see the actual raw data, add the *-v* flag to disable the display
helpers:

```nohighlight
$ choria rpc package status package=puppet-agent -I choria0.choria -v
.
.
choria0.choria                          : OK
   {
      "arch": "x86_64",
      "ensure": "6.17.0-1.el7",
      "epoch": "0",
      "name": "puppet-agent",
      "output": null,
      "provider": "yum",
      "release": "1.el7",
      "version": "6.17.0"
   }
```


This data can also be returned in JSON format:

```nohighlight
$ choria rpc package status package=puppet-agent -I choria0.choria -j
{
   "agent": "package",
   "action": "status",
   "replies": [
      {
         "sender": "choria0.choria",
         "statuscode": 0,
         "statusmsg": "OK",
         "data": {
            "arch": "x86_64",
            "ensure": "6.17.0-1.el7",
            "epoch": "0",
            "name": "puppet-agent",
            "output": null,
            "provider": "yum",
            "release": "1.el7",
            "version": "6.17.0"
         }
      }
   ],
   "request_stats": {
      "requestid": "4df150d6aa304e969e0380f82979a4b2",
      "no_responses": [],
      "unexpected_responses": [],
      "discovered": 1,
      "failed": 0,
      "ok": 1,
      "responses": 1,
      "publish_time": 264426137,
      "request_time": 2705429581,
      "discover_time": 309677895,
      "start_time_utc": "2020-12-29T19:24:12.017857117Z"
   },
   "summaries": {
      "arch": {
         "x86_64": 1
      },
      "ensure": {
         "6.17.0-1.el7": 1
      }
   }
}
```

## Filtering results

When using the *rpc* application you can perform additional filtering of the results, this will include only returned results if they matches a filter. This is a pure client side filter, all the data still has to arrive at the client and be processed before being removed. 

The filter is using the [expr](https://github.com/antonmedv/expr) language to match against an individual reply. And we support [GJSON Path Syntax](https://github.com/tidwall/gjson/blob/master/SYNTAX.md) for accessing data in the reply.

If we wanted to remove all failing results, and those where Puppet is idling from a Puppet Status query we can do this by combining these:

```nohighlight
$ choria rpc puppet status --filter-replies 'ok() && data("idling")' -j
Discovering nodes using the choria method ....27

27 / 27    3s [====================================================================] 100%

example.net
         Applying: false
   Daemon Running: true
     Lock Message:
          Enabled: true
           Idling: true
         Last Run: 1610223559
          Message: Currently idling; last completed run 25 minutes 03 seconds ago
   Since Last Run: 1503
           Status: idling

Summary of Applying:

   false: 1

Summary of Idling:

   true: 1

Summary of Status:

   idling: 1

Finished processing 27 / 27 hosts in 5.672902928s
```

The query `ok() && data("idling")` is checking the individual reply status code based on the table below and then accessing the `idling` data item in the result - a boolean.

There is a few things to note here.

 * 27 nodes were discovered, and we received 27 replies
 * only 1 matched the filter
 * the reply summaries consider only those replies that were not filtered out

These kinds of queries have to always return a boolean value.

## Request Chaining

{{% notice tip %}}
This is an experimental feature and subject to change
{{% /notice %}}

Built on the previous feature you can use the result - optionally filtered - of one RPC request as discovery source for another.

Lets say we want to query our fleet for all the nodes where a specific version of a package is installed and then do a package upgrade request on just those nodes.

```nohighlight
$ choria rpc package status package="zsh" -j --filter-replies 'ok() && data("version")=="4.3.11"' | choria req package update package=zsh
```

Here we do a `package#status` query on the `zsh` package, we select all ok replies where the version is `4.3.11` and then do a `package#update` on the matching nodes.

You can mix and match, for example you can do a service restart on all machine matching a specific package etc.

## Exit codes

When *mco* or *choria* finishes, it generates an exit code. The returned exit code depends on the nature of the issue:

| Exit Code | Description                                                                                 |
|-----------|---------------------------------------------------------------------------------------------|
| 0         | If nodes were discovered and all passed.                                                    |
| 0         | If no discovery was performed, at least 1 response was received, and all responses were OK. |
| 1         | If no nodes were discovered, or if mco encountered an issue not listed here                 |
| 2         | If nodes were discovered but some RPC requests failed.                                      |
| 3         | If nodes were discovered, but no responses were received.                                   |
| 4         | If no discovery was performed, and no responses were received.                              |

## Custom Applications

The *rpc* application should suit most needs. However, sometimes the
data being returned calls for customization such as custom aggregation,
summarising or complete custom display.

In such cases, a custom application may be useful For example, the
*package* application provides concluding summaries and provides some
basic safe guards for its use. The agent also provides the commonly
required data. Typical *package* output looks like this:

```nohighlight
$ mco service restart sshd
Do you really want to operate on services unfiltered? (y/n): y

 * [ ============================================================> ] 3 / 3


Summary of Service Status:

   running = 3


Finished processing 3 / 3 hosts in 537.23 ms
```


Notice how this application recognizes that you are acting on all
possible machines, an action which might have a big impact on your
servers. Consequently, *service* prompts for confirmation and, at the
end of processing, displays a brief summary.

While the behaviors of custom applications are not always consistent
with each other, in general they accept the standard discovery flags.
For details of which flags are accepted in a given application, use the
*mco help appname* command.

To discover which custom applications are available,  run *mco* or *mco
help*.
