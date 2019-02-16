+++
title = "Basic Playbook"
weight = 400
+++

Lets consider a basic Playbook used to restart Puppet Server and PuppetDB in a clean safe way.  Restarting Puppet Server is quite involved as you want to ensure not to interrupt the normal operations of things, so first we gracefully stop all the agents that might currently be using Puppet and then restart Puppet Server + PuppetDB followed by a Puppet run orchestrated in batches.

{{% notice tip %}}
If you just want to dive in and get your hands dirty review the [Tips and Patterns](../tips/) section which explains where to place Playbooks and show you much more detail
{{% /notice %}}

In details we will:

 * Expect a *cluster* as input which it will use to limit the discovery of nodes to some subset
 * Notify Slack that a restart is about to begin
 * Find nodes with *roles::puppetserver* class using PuppetDB PQL and mark those for upgrade
 * Find nodes with the *puppet* agent on them using PuppetDB PQL and mark those as managed nodes
 * Validate the versions of the *puppet* and *service* agents
 * Disable Puppet Agent on all the managed nodes
 * Wait for Puppet to finish doing in-progress Puppet catalog runs
 * Notify *graphite* about the server restart being done
 * Stop the *puppetserver* service
 * Restart the *puppetdb* service
 * Start the *puppetserver* service
 * Walk the managed nodes in groups of 10 and enable them, run puppet without splay and wait for them to finish
 * Notify slack that the process was completed

On failure further slack notifications will be sent.

To complete this task we write 2 playbooks, one that does the work without any error handling and one to run the other with the Slack error and success handling.

This playbook is on purpose verbose, in reality you would make smaller playbooks and re-using them or using Puppet functions to make small utilities to do common tasks - like we did with the `example::slack` one here.

```puppet
plan example::restart_puppetserver_no_error_handling (
  Enum[alpha, bravo] $cluster
) {
  # Discover nodes using `choria` method which uses PuppetDB PQL
  $puppet_servers = choria::discover("mcollective",
    "discovery_method" => "choria",
    "classes"  => ["roles::puppetserver"],
    "facts" => ["cluster=${cluster}"],
    "at_least" => 1,
    "uses" => { "service" => ">= 3.1.5" },
    "when_empty" => "Could not find any Puppet Servers to restart"
  )

  $puppet_agents = choria::discover("mcollective",
    "discovery_method" => "choria",
    "agents" => ["puppet"],
    "facts" => ["cluster=${cluster}"],
    "at_least" => 1,
    "uses" => { "puppet" => ">= 1.13.1" }
  )

  # Disable all the matched Puppet Agents
  choria::task(
    "action" => "puppet.disable",
    "nodes" => $puppet_agents,
    "fail_ok" => true,
    "silent" => true,
    "properties" => {"message" => "restarting puppet server"}
  )

  # Wait for them to sleep up to 200 seconds
  choria::task(
    "action"    => "puppet.status",
    "nodes"     => $puppet_agents,
    "assert"    => "idling=true",
    "tries"     => 10,
    "silent"    => true,
    "try_sleep" => 20,
  )

  # Notify graphite via a it's event API
  choria::task("graphite_event",
    "description" => "Restarting Puppet Server",
    "what" => "playbook event",
    "data" => "cluster: ${cluster}",
    "graphite" => "https://graphite.example.net/events/",
    "tags" => ["puppet", "playbooks", $cluster]
  )

  # Stop puppet server
  choria::task(
    "action" => "service.stop",
    "nodes"     => $puppet_servers,
    "properties" => {
      "service" => "puppetserver"
    },
  )

  # Restart puppetdb
  choria::task(
    "action" => "service.restart",
    "nodes"     => $puppet_servers,
    "properties" => {
      "service" => "puppetdb"
    },
  )

  # Start puppet server
  choria::task(
    "action" => "service.start",
    "nodes"     => $puppet_servers,
    "properties" => {
      "service" => "puppetserver"
    },
  )

  # Loop the Puppet Agents in groups of 10 and enable, run once and wait on each group
  $puppet_agents.choria::in_groups_of(10) |$nodes| {
    choria::task(
      "action" => "puppet.enable",
      "nodes" => $nodes,
      "silent" => true
    )

    choria::task(
      "action" => "puppet.runonce",
      "nodes" => $nodes,
      "fail_ok" => true,
      "properties" => {
        "force" => true
      }
    )

    choria::task(
      "action"    => "puppet.status",
      "nodes"     => $nodes,
      "assert"    => "idling=true",
      "tries"     => 10,
      "silent"    => true,
      "try_sleep" => 20,
      "pre_sleep" => 10,
    )
  }

  # Pass the list of Puppet Servers back to the caller or CLI
  $puppet_servers
}
```

We now add a wrapper plan that does error handling and notifies slack etc:

```puppet
plan example::restart_puppetserver (
  Enum[alpha, bravo] $cluster
) {
  $slack_defaults = {
    "channel" => "#ops",
    "mood" => "good"
  }

  # Notify slack using a helper playbook
  choria::run_playbook("example::slack",
    $slack_defaults + {"message" => "Starting a Puppet Server restart process"})

  # Call the above playbook and prevent it from raising an exception
  $servers = choria::run_playbook("example::restart_puppetserver_no_error_handling",
    _catch_errors => true,
    "cluster" => $cluster
  )

  # If the above playbook failed this will be an error object that we handle here
  $servers.choria::on_error |$err| {
    choria::run_playbook("example::slack",
      $slack_defaults + {
        "message" => sprintf("Restaring Puppet Server failed: %s", $err.message),
        "mood" => "bad"
      })

    fail($err.message)
  }

  # When it did not fail it would just be the servers list, so we notify slack what we did
  choria::run_playbook("example::slack",
    $slack_defaults + { "message" => sprintf("Restarted %s Puppet Server on %s", $cluster, $servers.join(", ")) })

  $servers
}
```

The Slack helper plan is shown below, it uses a data binding to store settings like the API Keys in `~/.plans.rc`.  You should store `slack.token: your-secret-token` in `~/.plans.rc`.

```puppet
plan acme::slack (
  String $message,
  String $channel = "#general",
  Enum[good, bad] $mood = "good",
) {
  # file store to keep token secret
  $token = choria::data("slack.token",
    "type"   => "file",
    "file"   => "~/.plans.rc",
    "format" => "yaml"
  )

  unless $token {
    fail("A slack API token is needed")
  }

  choria::task("slack",
    "token"   => $token,
    "channel" => $channel,
    "text"    => $message,
    "color"   => $mood
  )
}
```

You can run this playbook through the CLI, but lets look at help first, you can see our inputs are provided via *--cluster* or as json.

```bash
$ mco playbook run example::restart_puppetserver --modulepath modules --help

Choria Playbook Runner

Usage:   mco playbook [OPTIONS] <ACTION> <PLAYBOOK>

  The ACTION can be one of the following:

    run       - run the playbook as your local user

  The PLAYBOOK is a YAML file or Puppet Plan describing the
  tasks

  Passing --help as well as a PLAYBOOK argument will show
  flags and help related to the specific playbook.

  Any inputs to the playbook should be given on the CLI.

  A report can be produced using the --report argument
  when running a playbook

Application Options
        --cluster CLUSTER            Plan input property (Enum['alpha', 'bravo'])
        --input INPUT                JSON input to pass to the task
        --modulepath PATH            Path to find Puppet module when using the Plan DSL
        --loglevel LEVEL             Override the loglevel set in the playbook (debug, info, warn, error, fatal)
    -c, --config FILE                Load configuration from file rather than default
    -v, --verbose                    Be verbose
    -h, --help                       Display this screen

The Marionette Collective 2.11.4
```

And you can run the playbook using a few methods:

```
mco playbook run example::restart_puppetserver --modulepath modules --cluster alpha
```

Some effort is made to convert from the CLI to Puppet data types but for complex inputs you will have to use a JSON input

```
mco playbook run example::restart_puppetserver --modulepath modules --input '{"cluster":"alpha"}'
```

or with larger inputs from a file:

```
mco playbook run example::restart_puppetserver --modulepath modules --input @pb_input.json
```
