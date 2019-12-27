+++
title = "CLI Interaction Model"
weight = 203
toc = true
icon = "<b>2. </b>"
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

## Basic Usage of the *mco* Command

A simple example of a *mco* command can be seen below:

```nohighlight
$ mco ping
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
$ mco rpc --help
Generic RPC agent client application

Usage: mco rpc [options] [filters] --agent <agent> --action <action> [--argument <key=val> --argument ...]
Usage: mco rpc [options] [filters] <agent> <action> [<key=val> <key=val> ...]

Application Options
        --no-results, --nr           Do not process results, just send request
    -a, --agent AGENT                Agent to call
        --action ACTION              Action to call
        --arg, --argument ARGUMENT   Arguments to pass to agent

RPC Options
        --np, --no-progress          Do not show the progress bar
    -1, --one                        Send request to only one discovered nodes
        --batch SIZE                 Do requests in batches
        --batch-sleep SECONDS        Sleep time between batches
        --limit-seed NUMBER          Seed value for deterministic random batching
        --limit-nodes, --ln, --limit COUNT
                                     Send request to only a subset of nodes, can be a percentage
    -j, --json                       Produce JSON output
        --display MODE               Influence how results are displayed. One of ok, all or failed
    -c, --config FILE                Load configuration from file rather than default
    -v, --verbose                    Be verbose
    -h, --help                       Display this screen

Common Options
    -T, --target COLLECTIVE          Target messages to a specific sub collective
        --dt, --discovery-timeout SECONDS
                                     Timeout for doing discovery
    -t, --timeout SECONDS            Timeout for calling remote agents
    -q, --quiet                      Do not be verbose
        --ttl TTL                    Set the message validity period
        --reply-to TARGET            Set a custom target for replies
        --dm, --disc-method METHOD   Which discovery method to use
        --do, --disc-option OPTION   Options to pass to the discovery method
        --nodes FILE                 List of nodes to address
        --publish_timeout TIMEOUT    Timeout for publishing requests to remote agents.
        --threaded                   Start publishing requests and receiving responses in threaded mode.
        --sort                       Sort the output of an RPC call before processing.
        --connection-timeout TIMEOUT Set the timeout for establishing a connection to the middleware

Host Filters
    -W, --with FILTER                Combined classes and facts filter
    -S, --select FILTER              Compound filter combining facts and classes
    -F, --wf, --with-fact fact=val   Match hosts with a certain fact
    -C, --wc, --with-class CLASS     Match hosts with a certain config management class
    -A, --wa, --with-agent AGENT     Match hosts with a certain agent
    -I, --wi, --with-identity IDENT  Match hosts with a certain configured identity

The Marionette Collective 2.12.1
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
$ mco rpc service restart service=sshd
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
$ mco rpc service stop service=sshd
```

This shows the basic make-up of an RPC command. In this case we are:

 * using the *rpc* application - a generic application that can interact with any agent
 * directing our request to machines with the *service* agent
 * sending a request to the *stop* action of the service agent
 * supplying a value, *sshd*, to the *service* argument of the *stop* action

The same command has a longer form as well:

```nohighlight
$ mco rpc --agent service --action stop --argument service=sshd
```

These two commands are functionally identical.

### Discovering Available *Agents*

The above command showed you how to interact with the *service* agent,
but how can you find out that this agent even exists? On a correctly
installed Choria system you can use the *plugin* application to get
a list:

```nohighlight
$ mco plugin doc
Please specify a plugin. Available plugins are:

Agents:
  bolt_tasks                Downloads and runs Puppet Tasks
  choria_util               Choria Utilities
  filemgr                   File Manager
  nettest                   Perform network tests from a mcollective host
  package                   Manage Operating System Packages
  process                   Manages Operating System Processes
  puppet                    Manages the Life Cycle of the Puppet Agent
  rpcutil                   General helpful actions that expose stats and internals to SimpleRPC clients
  service                   Manages Operating System Services
  shell                     Run commands with the local shell
```

The first part of this list shows all the agents this computer is aware
of. In order to show up on this list, an agent must have a *DDL* file
and be installed locally.

To find out the *actions*, *inputs* and *outputs* for a specific agent
use the plugin application again:

```nohighlight
$ mco plugin doc agent/service
service
=======

Manages Operating System Services

      Author: R.I.Pienaar <rip@devco.net>
     Version: 4.0.1
     License: Apache-2.0
     Timeout: 60
   Home Page: https://github.com/choria-plugins/service-agent

   Requires MCollective 2.2.1 or newer

ACTIONS:
========
   restart, start, status, stop

   status action:
   --------------
       Gets the status of a service

       INPUT:
           service:
              Description: The service to get the status for
                   Prompt: Service Name
                     Type: string
                 Optional: false
               Validation: service_name
                   Length: 90


       OUTPUT:
           status:
              Description: The status of the service
               Display As: Service Status
            Default Value: unknown
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
$ mco rpc service status service=sshd
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

### Basic Filters

A key capability of Choria is fast discovery of network resources.
Discovery rules are written using *filters*.  For example:

```nohighlight
$ mco rpc service status service=httpd -S "environment=development or customer=acme"
```

This shows a filter rule that limits the RPC request to being run on
machines that are either in the Puppet environment *development* or
belong to the Customer *acme*.

Filtering can be based on *facts*, the presence of a *Configuration
Management Class* on the node, the node's *Identity*, or installed
*Agents* on the node.

Here are a number of examples of this with short descriptions of each
filter:

```nohighlight
# all machines with the service agent
$ mco ping -A service
$ mco ping --with-agent service

# all machines with the apache class on them
$ mco ping -C apache
$ mco ping --with-class apache

# all machines with a class that match the regular expression
$ mco ping -C /service/

# all machines in the UK
$ mco ping -F country=uk
$ mco ping --with-fact country=uk

# all machines in either UK or USA
$ mco ping -F "country=/uk|us/"

# just the machines called dev1 or dev2
$ mco ping -I dev1 -I dev2

# all machines in the domain foo.com
$ mco ping -I /foo.com$/
```

As you can see, you can filter by Agent, Class and/or Fact, and you can
use regular expressions almost anywhere.  You can also combine filters
additively in a command so that all the criteria have to be matched.

Note: You can use a shortcut to combine Class and Fact filters:

```nohighlight
# all machines with classes matching /apache/ in the UK
$ mco ping -W "/apache/ location=uk"
```

### Complex *Compound* or *Select* Queries

While the above examples are easy to enter, they are limited in that
they can only combine filters additively. If you want to create searches
with more complex boolean logic use the *-S* switch. For example:

```nohighlight
$ mco ping -S "((customer=acme and environment=staging) or environment=development) and /apache/"
```

The above example shows a scenario where the development environment is
usually labeled *development* but one customer has chosen to use
*staging*. You want to find all machines in those customers'
environments that match the class *apache*. This search would be
impossible using the previously shown methods, but the above command
uses *-S* to allow the use of boolean operators such as *and* and *or*
so you can easily build the logic of the search.

The *-S* switch also allows for negative matches using *not* or *!*:

```nohighlight
$ mco ping -S "environment=development and !customer=acme"
$ mco ping -S "environment=development and not customer=acme"
```

### Filtering Using Data Plugins

Custom data plugins can also be used to create complex filters:

```nohighlight
$ mco ping -S "fstat('/etc/hosts').md5=/baa3772104/ and environment=development"
```

This will search for the md5 hash of a specific file with matches
restricted to the *development* environment.  Note that as before,
regular expressions can also be used.

As with agents, you can also discover which plugins are available for
use:

```nohighlight
$ mco plugin doc

Please specify a plugin. Available plugins are:

Agents:
  .
  .

Data Queries:
  agent                     Meta data about installed MColletive Agents
  bolt_task                 Information about past Bolt Task
  collective                Collective membership
  fact                      Structured fact query
  fstat                     Retrieve file stat data for a given file
  nettest                   Checks if connecting to a host on a specified port is possible
  package                   Checks the status of a package
  process                   Checks if a process matching a supplied pattern is present
  puppet                    Information about Puppet agent state
  resource                  Information about Puppet managed resources
  service                   Checks the status of a service
```

For information on the input these plugins take and  output they provide
use the *mco plugin doc fstat* command.

Currently, each data function can only accept one input while matches
are restricted to a single output field per invocation.

## Chaining RPC Requests

The *rpc* application can chain commands one after the other. The
example below uses the *package* agent to find machines with a specific
version of mcollective and then schedules Puppet runs on those machines:

```nohighlight
$ mco rpc package status package=mcollective -j|jgrep "data.properties.ensure=2.0.0-6.el6" |mco rpc puppetd runonce
```

Choria results can also be filtered using the opensource gem,
jgrep. Choria data output is fully compatible with jgrep.

## Using with PuppetDB

Recent versions of PuppetDB has a built in query language called
Puppet Query Language that you use via the `puppet query` command.

Much like the above example of chaining RPC requests Choria supports
reading results from Puppet Query:

```nohighlight
$ puppet query "inventory { facts.os.name = 'CentOS' }"| mco rpc puppetd runonce
```

This will run Puppet on all CentOS machines

Additionally Choria includes a discovery plugin that can communicate directly
with PuppetDB for you so you do not need to learn PQL.  Read about this plugin
in the Optional Configuration section.

## Seeing the Raw Data

By default the *rpc* application will try to show human-readable data.
To see the actual raw data, add the *-v* flag to disable the display
helpers:

```nohighlight
$ mco rpc package status package=puppet-agent -I choria0.choria -v
.
.
choria0.choria                          : OK
    {:output=>nil,     :epoch=>"0",     :arch=>"x86_64",     :ensure=>"5.5.1-1.el7",     :version=>"5.5.1",     :provider=>"yum",     :name=>"puppet-agent",     :release=>"1.el7"}
```


This data can also be returned in JSON format:

```nohighlight
$ mco rpc package status package=puppet-agent -I choria0.choria -j
[
  {
    "agent": "package",
    "action": "status",
    "sender": "choria0.choria",
    "statuscode": 0,
    "statusmsg": "OK",
    "data": {
      "output": null,
      "epoch": "0",
      "arch": "x86_64",
      "ensure": "5.5.1-1.el7",
      "version": "5.5.1",
      "provider": "yum",
      "name": "puppet-agent",
      "release": "1.el7"
    }
  }
]
```

## Error Messaging

When an application encounters an error, it returns an explanatory
string:

```nohighlight
% mco rpc rpcutil foo
rpc failed to run: Attempted to call action foo for rpcutil but it's not declared in the DDL (MCollective::DDLValidationError)
```

By default only an abbreviated error string is shown that  provides some
insight into the nature of the problem.  For more details, add the *-v*
flag to show a full stack trace:

```nohighlight
% mco rpc rpcutil foo -v
rpc failed to run: Attempted to call action foo for rpcutil but it's not declared in the DDL (MCollective::DDLValidationError)
        from /usr/lib/ruby/site_ruby/1.8/mcollective/ddl.rb:303:in `validate_rpc_request'
        from /usr/lib/ruby/site_ruby/1.8/mcollective/rpc/client.rb:218:in `method_missing'
        from /home/rip/.mcollective.d/lib/mcollective/application/rpc.rb:133:in `send'
        from /home/rip/.mcollective.d/lib/mcollective/application/rpc.rb:133:in `main'
        from /usr/lib/ruby/site_ruby/1.8/mcollective/application.rb:283:in `run'
        from /usr/lib/ruby/site_ruby/1.8/mcollective/applications.rb:23:in `run'
        from /usr/bin/mco:20
```

## Exit codes

When *mco* finishes, it generates an exit code. The returned exit code depends on the nature of the issue:

|Exit Code|Description|
|---------|-----------|
|0|If nodes were discovered and all passed.|
|0|If no discovery was performed, at least 1 response was received, and all responses were OK.|
|1|If no nodes were discovered, or if mco encountered an issue not listed here|
|2|If nodes were discovered but some RPC requests failed.|
|3|If nodes were discovered, but no responses were received.|
|4|If no discovery was performed, and no responses were received.|

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
