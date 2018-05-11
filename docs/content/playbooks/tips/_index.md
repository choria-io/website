+++
title = "Tips and Patterns"
weight = 499
+++

## Module Layout and Naming

Generally the Playbooks behave as you would expect any modern Puppet function or data type.

You place the Playbook *mymod::mybook* in *modules/mymod/plans/mybook.pp* and in there you need *plan mymod::mybook*. Naming of Playbooks are subject to the same naming rules as Puppet, for example you cannot have a *-* in the Playbook name.

The modules with playbooks all go into a directory and you have to point at that directory with *--modulepath*. which defaults to *~/.puppetlabs/etc/code/modules*, the same location *puppet module install* will place modules by default.

When running in this mode Puppet has a number of changes in behavior for example strict variable checks are enabled by default and templates are disabled.  I cannot find a list of these behavior changes but it's something to be aware of.

## Interacting With Task Results

When a task is run it returns an instance of *Choria::TaskResults* which contains one of more *Choria::TaskResult* for every node that was affected by the task.

Each *Choria::TaskResult* has the following properties you can use:

|Property / Method|Type|Description|
|-----------------|----|-------|
|host             |Choria::Node   |The hostname this result apply to|
|result           |Data           |The data that was returned from the task, this depends on the type of task|
|error            |Optional[Error]|A Puppet standard Error object if the task failed|
|ok               |Boolean        |If the task was successful, always true when `fail_ok` is given|
|type             |String[1]      |The type of task that this result represents, like *mcollective*|
|*$r["foo"]*      |Data           |Utility to access the data of the result if it's a hash|
|value            |Data           |The entire value that the task produced|

Each *Choria::TaskResults* has the following properties you can use:

|Property / Method|Type|Description|
|-------------------|----|-------|
|results            |Array[Choria::TaskResult]|The individual results|
|count              |Integer                  |How many results are contained in this set|
|empty              |Boolean                  |If no results are contained in this set|
|error_set          |Choria::TaskResults      |A new result set with only failing nodes|
|ok_set             |Choria::TaskResults      |A new result set with only passing nodes|
|ok                 |Boolean                  |If all contained results were ok|
|fail_ok            |Boolean                  |If failures are ignored when checking ok|
|hosts              |Choria::Nodes            |List of nodes in this result set|
|message            |String                   |A short descriptive message about the outcome|
|*find("some.node")*|Optional[Choria::TaskResult]|Finds the Choria::TaskResult for a specific node|
|first              |Optional[Choria::TaskResult]|Returns just the first Choria::TaskResult|

With these you can do complex error and result handling, display statuses you like, send to Slack etc.

```puppet
$result = choria::task("action" => "rpcutil.ping", "nodes" => $nodes, "fail_ok" => true)

if $result.error_set.empty {
  notice(sprintf("Ping reached %d nodes", $result.count))
  notice(sprintf("Outcome: %s", $result.message))
} else {
  notice(sprintf("Ping failed on: %s", $result.error_set.hosts.join(", "))
}
```

If you return these from a plan - or the last statement in your plan is a *choria::task* you can interact and error handle plans based on these.

## Nodes on the CLI

Generally nodes come from node sets but you can also use a input to mimic a node set to some extend.

```puppet
plan example::cli_nodes (
  Array[String] $nodes
) {
  # use $nodes
}
```

You can now supply nodes like this:

```
$ mco playbook run playbook.yaml --nodes web1.example.net --nodes web2.example.net
```

You can also load nodes from some file like this:

```
$ mco playbook run playbook.yaml --input @nodes.json
```

Your JSON would just have a array of node names in JSON format, you can also use YAML format by naming the file *nodes.yaml*

## Error Handling Strategies

By default when you run a playbook any failure will just fail the playbook, but you might want to handle failures and branch to recovery code after failure, here's a Playbook that handles failures and success, I'll show a few possible syntaxes of the same thing:

The key to error handling is to pass `_catch_errors => true` to *choria::run_playbook* which would avoid raising errors instead giving you a chance to handle them.

```puppet
choria::run_playbook("example::app_upgrade", _catch_errors => true,
  "action" => "appmgr.restart",
  "nodes" => $nodes)

    .choria::on_error |Choria::TaskResults $err| {
      notice("Application upgrade failed: ${err.message}, recovering using example::recover")

      choria::run_playbook("example::rocover",
        "cluster" => $cluster
      )

      fail("deploying cluster ${cluster} failed, recovery run succesfully")}

    .choria::on_success |Choria::TaskResults $results| {
      notice("deployment succesful on cluster ${cluster}")}
```

This syntax might be a bit foreign to Puppet users, here's another approach but now you need temporary variables which can be very annoying:

```puppet
$result = choria::task("mcollective", _catch_errors => true,
  "action" => "appmgr.restart",
  "nodes" => $nodes
)

$result.choria::on_error |$results| {
  choria::run_playbook("example::rocover",
    "cluster" => $cluster
  )

  fail("deploying cluster ${cluster} failed, recovery run succesfully")
}

$result.choria::on_success |$results| {
  notice("deployment succesful on cluster ${cluster}")
}
```

The Base Example page shows this in action where one Playbook wraps another and provides custom error handling.

These functions are clever enough to only trigger for *Choria::TaskResults* so if you wish to handle failures AND return non *Choria::TaskResults* from a Plan that's totally ok.

```puppet
plan example::update {
  $nodes = choria::discover(
    # ....
  )

  choria::task(
    # ...
  )

  $nodes
}
```

```puppet
$nodes = choria::run_playbook("example::update", _catch_errors => true)
  .on_error |$err| {
    # handle
  }

notice($nodes.join(", "))
```

In this example your *$nodes* is an array or strings, the *example::update* returns the nodes and you print it using *join* which only works with arrays.  However if the task failed you'd get a *Choria::TaskResults* back and the error handling will deal with that failure for you.

## Utility Functions and Playbooks

These playbooks tend to be made up of a lot of basic building blocks.  For example I find I often need to disable Puppet and wait for them to idle, lets look how we can make this reusable.

```puppet
plan example::puppet::disable_and_wait (
  String $message = sprintf("Disabled by Choria Playbook $s", $facts["choria"]["playbook"]),
  Integer $checks = 20,
  Integer $sleep = 10
){
  $nodes = choria::discover(
    "discovery_method" => "choria",
    "test" => true,
    "agents" => ["puppet"],
  )

  choria::task(
    "action" => "puppet.disable",
    "nodes" => $nodes,
    "fail_ok" => true,
    "silent" => true,
    "properties" => {
      "message" => $message
    }
  )

  choria::task(
    "action"    => "puppet.status",
    "nodes"     => $nodes,
    "assert"    => "idling=true",
    "tries"     => $checks,
    "silent"    => true,
    "try_sleep" => $sleep,
    "pre_sleep" => 5,
  )
}
```

You can now use this Playbook wherever you need this functionality or indeed from the CLI.  This is the benefit of using a Playbook as the abstraction rather than functions since they can be called from the CLI.

```puppet
$results = choria::run_playbook("example::puppet::disable_and_wait",
  "message" => "Disabled while restarting Puppet Server",
)
```

```nohighlight
mco playbook run example::puppet::disable_and_wait --message "disabled while testing"
```

For things that clearly have no standalone value functions can be made:

```puppet
function example::plan_rc(String $key) >> String {
  choria::data($key, {
    "type"   => "file",
    "file"   => "~/.plans.rc",
    "format" => "yaml"
  })
}
```

Here you can just call `$something = example::plan_rc("something")` whenever you want to fetch some data from your data source.

## Iterating Subsets of Node Sets

There's a small utility function that can iterate over arrays and call a provided block with a subset of the whole array, this is handy for performing multiple actions on batches.

```puppet
$nodes.choria::in_groups_of(10) |$n| {
  # here $n is 10 nodes or fewer
  # see the example in the basics section for this in action
}
```

## Validating Playbook syntax

You can use the standard Puppet CLI to validate your Playbook syntax if you pass *--tags* like `puppet parser validate --tasks playbook.pp`

## Documentation using Puppet Strings

You can document your plans using the normal Puppet Strings, when generating documents for a module with Playbooks in it just pass *--tags* like `puppet strings generate --tasks 'example/**/*.pp'`
