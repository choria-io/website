+++
title = "Ruby Agents"
weight = 20
+++

MCollective RPC works because it makes a lot of assumptions about how you write agents, we'll try to capture those assumptions here and show you how to apply them to our Echo agent.

{{% notice warning %}}
This section of the documentation have recently been migrated from the old mcollective documentation, we are still in the process of verifying every example works in modern mcollective.  If you find any issues please get in touch.
{{% /notice %}}

## Conventions regarding Incoming Data

As you've seen in the [client documentation](/development/mcorpc/clients/) our clients will send requests like:

```ruby
mc.echo(:msg => "Welcome to MCollective RPC")
```

A more complex example might be:

```ruby
exim.setsender(:msgid => "1NOTVx-00028U-7G", :sender => "foo@bar.com")
```

Effectively this creates a hash with the members _:msgid_ and _:sender_ and invokes the _setsender_ action.

You cannot use the a data item called _:process\_results_ as this has special meaning to the agent and client.  This will indicate to the agent that the client isn't going to be waiting to process results.  You might choose not to send back a reply based on this.

## Sample Agent

Here's our sample _Echo_ agent:

```ruby
# $libdir/mcollective/agent/echo.ddl
metadata :name        => "echo",
         :description => "Echo service for MCollective",
         :author      => "R.I.Pienaar",
         :license     => "GPLv2",
         :version     => "1.1",
         :url         => "https://docs.puppetlabs.com/mcollective/simplerpc/agents.html",
         :timeout     => 60

action "echo", :description => "Echos back any message it receives" do
    display :always

    input :msg,
          :prompt      => "Message",
          :description => "Your message",
          :type        => :string,
          :validation  => ".*",
          :optional    => false,
          :maxlength   => 1024

    output :msg,
        :description => "Your message",
        :display_as  => "Message",
        :default     => ""
end
```


```ruby
# $libdir/mcollective/agent/echo.rb
module MCollective
  module Agent
    class Echo<RPC::Agent
      # Basic echo server
      action "echo" do
        reply[:msg] = request[:msg]
      end
    end
  end
end
```

You'll notice it comes in two parts, the agent definition in the DDL file, and the implementation in the ruby class.

### Agent Name

The agent name is derived from the class name, the example code creates the ruby class *MCollective::Agent::Echo*, the agent name for this would be *echo*.


## Help and the Data Description Language

We have a file that goes together with an agent implementation which is used to describe the agent in detail.  This is referred to as the ddl file, _$libdir/mcollective/agent/echo.ddl_ in the previous example.

The DDL file can be used to generate help texts, and to declare the validation that arguments must pass.  This information helps us in making more robust clients and can be used to auto generate user interfaces.

We'll not say much more about the DDL here, for more details read the [DDL](../ddl/) reference.


## Actions

Actions are the individual tasks that your agent can do, we declare them in the DDL file, and put their implementation in the ruby class named for the agent (ie *MCollective::Agent::Echo* for the *echo* agent).

### Declaring Actions

Actions with their inputs and outputs are declared in the DDL for the agent.  From our echo example here's the declaration of the echo action again:

```ruby
# $libdir/mcollective/agent/echo.ddl - incomplete fragment
action "echo", :description => "Echos back any message it receives" do
    display :always

    input :msg,
          :prompt      => "Message",
          :description => "Your message",
          :type        => :string,
          :validation  => ".*",
          :optional    => false,
          :maxlength   => 1024

    output :msg,
        :description => "Your message",
        :display_as  => "Message",
        :default     => ""
end
```


### Implementing Actions

The implementation of an action is added to the agent class:

```ruby
# $libdir/mcollective/agent/echo.rb
module MCollective
  module Agent
    class Echo<RPC::Agent
      action "echo" do
        reply[:msg] = request[:msg]
      end
    end
  end
end
```

#### Accessing the Input

As you see from the echo example our input is easy to get to by just looking in *request*, this would be a Hash of exactly what was sent in by the client in the original request.

The request object is in instance of *MCollective::RPC::Request*, you can also gain access to the following:

|Property|Description|
|--------|-----------|
|time|The time the message was sent|
|action|The action it is directed at|
|data|The actual hash of data|
|sender|The id of the sender|
|agent|Which agent it was directed at|

Since data is the actual Hash you can gain access to your input like:

```ruby
request.data[:msg]
```

Accessing via _request.data_ will give you full access to all the normal Hash methods, eg _request.data.keys_.

```ruby
request[:msg]
```

Accesing via _request[]_ will only give you access to _include?_ like behaviour, nil will be returned for unknown keys or data that is not supplied.

#### Running Shell Commands

A helper function exist that makes it easier to run shell commands and gain access to their _STDOUT_ and _STDERR_.

We recommend everyone use this method for calling to shell commands as it forces _LC_ALL_ to _C_ as well as wait on all the children and avoids zombies, you can set unique working directories and shell environments that would be impossible using simple _system_ that is provided with Ruby.

The simplest case is just to run a command and send output back to the client:

```ruby
reply[:status] = run("echo 'hello world'", :stdout => :out, :stderr => :err)
```

Here you will have set _reply[:out]_, _reply[:err]_ and _reply[:status]_ based on the output from the command.

You can append the output of the command to any string:

```ruby
out = []
err = ""
status = run("echo 'hello world'", :stdout => out, :stderr => err)
```

Here the STDOUT of the command will be saved in the variable _out_ and not sent back to the caller.  The only caveat is that the variables _out_ and _err_ should have the _<<_ method, so if you supplied an array each line of output will be a single member of the array.  In the example _out_ would be an array of lines while _err_ would just be a big multi line string.

By default any trailing new lines will be included in the output and error:

```ruby
reply[:status] = run("echo 'hello world'", :stdout => :out, :stderr => :err)
reply[:stdout].chomp!
reply[:stderr].chomp!
```

You can shorten this to:

```ruby
reply[:status] = run("echo 'hello world'", :stdout => :out, :stderr => :err, :chomp => true)
```

This will remove a trailing new line from the _reply[:out]_ and _reply[:err]_.

If you wanted this command to run from the _/tmp_ directory:

```ruby
reply[:status] = run("echo 'hello world'", :stdout => :out, :stderr => :err, :cwd => "/tmp")
```

Or if you wanted to include a shell Environment variable:

```ruby
reply[:status] = run("echo 'hello world'", :stdout => :out, :stderr => :err, :environment => {"FOO" => "BAR"})
```

The status returned will be the exit code from the program you ran, if the program completely failed to run in the case where the file doesn't exist, resources were not available etc the exit code will be -1

You have to set the cwd and environment through these options, do not simply call _chdir_ or adjust the _ENV_ hash in an agent as that will not be safe in the context of a multi threaded Ruby application.

### Actions in external scripts

Actions can also be implemented using other programming languages as long as they support JSON.

```ruby
action "test" do
  implemented_by "/some/external/script"
end
```

The script _/some/external/script_ will be called with 2 arguments:

 * The path to a file with the request in JSON format
 * The path to a file where you should write your response as a JSON hash

You can also access these 2 file paths in the _MCOLLECTIVE_REPLY_FILE_ and _MCOLLECTIVE_REQUEST_FILE_ environment variables

Simply write your reply as a JSON hash into the reply file.

The exit code of your script should correspond to the ones in [ResultsandExceptions][].  Any text in STDERR will be
logged on the server at _error_ level and used in the text for the fail text.

Any text to STDOUT will be logged on the server at level _info_.

These scripts can be placed in a standard location:

```ruby
action "test" do
  implemented_by "script.py"
end
```

This will search each configured libdir for _mcollective/agent/$agent_name/script.py_ and _agent/$agent_name/script.py_, and will use the former if found. If you specified a full path it will not try to find the file in libdirs.


## Constructing Replies

### Reply Data

The reply data is in the *reply* variable and is an instance of *MCollective::RPC::Reply*.

```ruby
reply[:msg] = request[:msg]
```

### Reply Status

Replies have a strong concept of success or failure indicating the overall success of the request, the table below shows the valid statusses:

|Status Code|Description|Exception Class|
|-----------|-----------|---------------|
|0|OK| |
|1|OK, failed.  All the data parsed ok, we have a action matching the request but the requested action could not be completed.|RPCAborted|
|2|Unknown action|UnknownRPCAction|
|3|Missing data|MissingRPCData|
|4|Invalid data|InvalidRPCData|
|5|Other error|UnknownRPCError|

```ruby
action "rmmsg" do
  validate :msg, String
  validate :msg, /[a-zA-Z]+-[a-zA-Z]+-[a-zA-Z]+-[a-zA-Z]+/
  reply.fail("No such message #{request[:msg]}", 1) unless have_msg?(request[:msg])

  # check all the validation passed before doing any work
  return unless reply.statuscode == 0

  # now remove the message from the queue
end
```

The number in _reply.fail_ corresponds to the codes in the table above it would default to _1_ so you could just say:

```ruby
reply.fail "No such message #{request[:msg]}" unless have_msg?(request[:msg])
```

This is hypothetical action that is supposed to remove a message from some queue, if we do have a String as input that matches our message id's we then check that we do have such a message and if we don't we fail with a helpful message.

Technically this will just set _statuscode_ and _statusmsg_ fields in the reply to appropriate values.

It won't actually raise exceptions or exit your action though you should do that yourself as in the example here.

There is also a _fail!_ instead of just _fail_ it does the same basic function but also raises exceptions.  This lets you abort processing of the agent immediately without performing your own checks on _statuscode_ as above later on.

## Sharing code between agents

Sometimes you have code that is needed by multiple agents or shared between the agent and client.  MCollective has name space called _MCollective::Util_ for this kind of code and the packagers and so forth supports it.

Create a class with your shared code given a name like _MCollective::Util::Yourco_ and save this file in the libdir in _util/yourco.rb_

A sample class can be seen here:

```ruby
module MCollective
  module Util
    class Yourco
      def dosomething
      end
    end
  end
end
```

You can now use it in your agent or clients by first loading it from the MCollective lib directories:

```ruby
MCollective::Util.loadclass("MCollective::Util::Yourco")

helpers = MCollective::Util::Yourco.new
helpers.dosomething
```

## Authorization

You can write a fine grained Authorization system to control access to actions and agents, please see [Choria AAA](/docs//configuration/aaa/) for full details.

## Auditing

The actions that agents perform can be Audited by code you provide, potentially creating a centralized audit log of all actions.  See [Choria AAA](/docs/configuration/aaa/) for full details.

## Logging

You can write to the server log file using the normal logger class:

```ruby
Log.debug("Hello from your agent")
```

You can log at levels _info_, _warn_, _debug_, _fatal_ or _error_.

## Processing Hooks

We provide a few hooks into the processing of a message, you've already used this earlier to set metadata.

You'd use these hooks to add some functionality into the processing chain of agents, maybe you want to add extra logging for audit purposes of the raw incoming message and replies, these hooks will let you do that.

Hook Function Name                        | Description
------------------------------------------|------------------------------------------------------------------------------------------------------
`startup_hook`                            | Called at the end of the initialize method of the _RPC::Agent_ base class
`before_processing_hook(msg, connection)` | Before processing of a message starts, pass in the raw message and the _MCollective::Connector_ class
`after_processing_hook`                   | Just before the message is dispatched to the client

### startup_hook

Called at the end of the _RPC::Agent_ standard initialize method.  Use this to adjust meta parameters, timeouts and any setup you need to do.

This will not be called right when the daemon starts up, we use lazy loading and initialization so it will only be called the first time a request for this agent arrives.

### before_processing_hook

Called just after a message was received from the middleware before it gets passed to the handlers.  _request_ and _reply_ will already be set, the msg passed is the message as received from the normal MCollective runner and the connection is the actual connector.

You can in theory send off new messages over the connector maybe for auditing or something, probably limited use case in simple agents.

### after_processing_hook

Called at the end of processing just before the response gets sent to the middleware.

This gets run outside of the main exception handling block of the agent so you should handle any exceptions you could raise yourself.  The reason  it is outside of the block is so you'll have access to even status codes set by the exception handlers.  If you do raise an exception it will just be passed onto the runner and processing will fail.

## Agent Configuration

You can save configuration for your agents in the main server config file:

```ini
plugin.helloworld.setting = foo
```

In your code you can retrieve the config setting like this:

```ruby
setting = config.pluginconf.fetch("helloworld.setting", "")
```

This will set the setting to whatever is in the config file or "" if unset.
