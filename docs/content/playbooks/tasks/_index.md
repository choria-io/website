+++
title = "Tasks"
weight = 430
toc = true
+++

Tasks are the main place you put things you want to happen.  Tasks are run using `choria::task` and would make up the main body of your Playbooks.

## Common Task options

There are some items that can be applied to any type of task:

|Option|Description|Sample|
|------|-----------|------|
|tries |On failure, try this many times|`tries => 10`|
|try_sleep|Sleep this long between tries|`try_sleep => 60`|
|pre_sleep|Sleep this long before running a task for the first try|`pre_sleep => 60`|
|fail_ok|A failure will be logged but not fail the playbook|`fail_ok => true`|

## MCollective task

```puppet
$result = choria::task("mcollective",
  "nodes" => $nodes,
  "action" => "puppet.disable",
  "batch_size" => 10,
  "silent" => true,
  "post" => ["summarize"],
  "properties" => {
    "message" => "Disabled by playbooks"
  }
)
```

|Option|Description|
|------|-----------|
|nodes|A node set to use, can be a template as here or just a hard coded array of nodes|
|action|A *agent* and *action* specified as one|
|batch_size|Optional setting to address the node set in smaller groups|
|batch_sleep_time|Optional setting that relates to `batch_size` and sets the sleep time between batches|
|silent|By default each result is logged, this disables that|
|post|Logs aggregate summaries if *summarize* - the only current valid entry - is given|
|properties|Any properties the action needs to function|
|assert|Any JGrep query that will be run over the RPC reply data|

### Asserting result state

The typical use case for asserting reply state is where you want to wait for a set of nodes to match a specific complex state.  You'd achieve this using fine grained assertions on the result data than the true/false that is typically used to determine success.

To use the `assert` feature you should have the `jgrep` gem installed in your Puppet Ruby on the node where you will run *mco playbook*.  You can use Hiera to do this:

```yaml
mcollective_choria::gem_dependencies:
  "jgrep": "1.5.0"
```

Some use cases:

  * Wait for all enabled Puppet nodes to stop runs before you make manual changes using other tasks
  * Dig into the result hashes and do complex boolean matches against the data such as asserting certain package version numbers from `package.status`

#### Examples:

This will wait for all `enabled` Puppet nodes to go into idling state for a period of time based on `tries` and `try_sleep`:

```puppet
choria::task("mcollective",
  "nodes" => $nodes,
  "action" => "puppet.status",
  "assert" => "idling=true and enabled=true",
  "pre_sleep" => 0,
  "tries" => 10,
  "try_sleep" => 30
)
```

An assertion like `summary.resources.failed_to_restart=0` against the `puppet.last_run_summary` action will dig deep into the result hash checking if any services failed.

You can create even more complex assertions like `applying=false and enabled=true and daemon_present=true` should you want to do complex matches.

Full details about the JGrep assertion language can be found on its site [jgrep.org](http://jgrep.org/).

On the CLI you can test these queries:

```
$ mco rpc puppet status -j | jgrep data.idling=true -s sender
```

This will show you the node names that match the above `idling` example.  Note you have to prefix the matching with `data` as per the JSON returned by `mco rpc -j`.

## Shell task

```puppet
choria::task("shell",
  "command" => "/home/rip/test.sh",
  "cwd" => "/tmp",
  "nodes" => $nodes,
  "environment" => {
    "HELLO" => "WORLD"
  }
)
```

{{% notice warning %}}
If you pass any inputs into your script arguments be sure to validate the input is safe for passing into a shell.  You can do this using the `Choria::ShellSafe` data type.
{{% /notice %}}

|Option|Description|
|------|-----------|
|command|The command to run, can be a full command with options specified and all|
|cwd|The directory will be the working directory for the command when run|
|nodes|A list of nodes, passed as *--nodes node1,node2* to your script|
|environment|A hash of any environment variables you wish to set|

## Webhook task

This allows arbitrary webhooks to be constructed and called via either GET or POST.

For a GET method the data gets URL encoded as request parts, for POST the data gets JSON encoded and sent.

```puppet
choria::task("webhook",
  "uri" => "https://hooks.example.net/webhook",
  "method" => "POST",
  "headers" => {
    "X-Token" => $token
  },
  "data" => {
    "message" => "Deployed Acme release ${version}",
    "nodes" => $nodes
  }
)
```

The request will include a header *X-Choria-Request-ID* with a unique UUID for every request.

|Option|Description|
|------|-----------|
|uri|The address to send the request to, supports *http* or *https*|
|method|Either *GET* or *POST*|
|headers|Hash of headers to send.  It will send a User Agent and Content Type on it's own but you can override those too here|
|data|Hash of data to send either as request arguments for GET or JSON encoded data for POST|

## Graphite Event Task

Sends a Graphite Event to any Graphite server. This is just a convenience wrapper around the _webhook_ task.

```puppet
choria::task("graphite_event",
  "what" => "playbook event",
  "data" => "app weather: ${release} cluster: ${cluster}",
  "graphite" => "https://graphite.example.net/events/",
  "tags" => ["weather", "deploy"]
)
```

|Option|Description|
|------|-----------|
|what  |Short description of the event|
|data  |Details about the event|
|graphite|Url to the _event_ endpoint|
|tags|Array of Strings that makes up tags about the event|
|headers|Hash of headers to send, same as for the webhook task|

## Slack task

Sends a message to a Slack channel, tested using the basic *bot user account* style bot.

```puppet
choria::task("slack",
  "token" => "xoxb-YOUR-TOKEN",
  "channel" => "#general",
  "text" => sprintf("Playbook %s completed", $facts["choria"]["playbook"])
)
```

The above will produce a slack notification like the one below.

![Slack Sample](../../slack-example.png)

This posts a message to Slack with your text using the *chat.postMessage* API, above is all the possible options.

|Option|Description|
|------|-----------|
|token|Your API token for the bot user|
|channel|The channel to send to|
|text|Markdown encoded text to send in the attachment|
|color|Color to use, web colors like *#aabbccdd* or one of *good*, *warning* or *danger*|
|username|The user to appear as on the slack channel|
