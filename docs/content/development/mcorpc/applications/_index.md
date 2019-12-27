+++
title = "Applications"
weight = 40
+++

## Overview

MCollective supports a single executable - called mco - and has a plugin type called application that lets you create applications for this single executable.

While we have a standard generic client called _mco rpc_ to interact with your network, sometimes this standard approach does not work - perhaps you wish to add specific input munging, query databases or just display the output in a custom manner.  Some agents have utility actions that produce so much data that just dumping it to the display is not useful.  Other actions are not designed for generic interaction - they support fuller automation flows.  For all these needs you can extend the _mco_ CLI with new plugins.

Interacting with the single executable system is documented in the [CLI Interaction Model](/docs/concepts/cli/) section.

Apart from the plumbing to hook these into the CLI you will have full access to the normal [MCollective Client](../clients/).

{{% notice warning %}}
This section of the documentation has recently been migrated from the old mcollective documentation, we are still in the process of verifying every example works in modern mcollective.  If you find any issues please get in touch.
{{% /notice %}}

## Basic Application

Applications goes in _libdir/mcollective/application/echo.rb_, the one below is a simple application that speaks to a hypothetical _echo_ action of a _helloworld_ agent. This agent has been demonstrated in : writing [agents][../agents/].

```ruby
class MCollective::Application::Echo<MCollective::Application
   description "Reports on usage for a specific fact"

   option :message,
          :description    => "Message to send",
          :arguments      => ["-m", "--message MESSAGE"],
          :type           => String,
          :required       => true

   def main
      mc = rpcclient("helloworld")

      printrpc mc.echo(:msg => configuration[:message], :options => options)

      printrpcstats
   end
end
```

Here's the application we wrote in action:

```nohighlight
$ mco echo
The message option is mandatory

Please run with --help for detailed help
```

```nohighlight
$ mco echo -m test

 * [ ============================================================> ] 1 / 1


example.com
   Message: test
      Time: Mon Jan 31 21:27:03 +0000 2011


Finished processing 1 / 1 hosts in 68.53 ms
```

Most of the techniques documented in MCollective [Clients](../clients/) can be reused here, we've just simplified a lot of the common used patterns like CLI arguments and incorporated it all in a single framework.

## Reference

### Usage Messages

To add custom usage messages to your application we can add lines like this:

```ruby
class MCollective::Application::Echo<MCollective::Application
   description "Reports on usage for a specific fact"

   usage "mco echo [options] --message message"
end
```

You can add several of these messages by just adding multiple such lines.

### Application Options

A standard options hash is available simply as options you can manipulate it and pass it into the RPC Client like normal. See the SimpleRPC [Clients] reference for more on this.

### CLI Argument Parsing

There are several options available to assist in parsing CLI arguments. The most basic option is:

```ruby
class MCollective::Application::Echo<MCollective::Application
   option :message,
          :description    => "Message to send",
          :arguments      => ["-m", "--message MESSAGE"]
end
```

In this case if the user used either _-m message_ or _--message message_ on the CLI the desired message would be in _configuration[:message]_

#### Required Arguments

You can require that a certain parameter is always passed:

```ruby
option :message,
  :description    => "Message to send",
  :arguments      => ["-m", "--message MESSAGE"],
  :required       => true
```

#### Argument data types

CLI arguments can be forced to a specific type, we also have some additional special types that the default ruby option parser can't handle on its own.

You can force data to be of type String, Integer etc:

```ruby
option :count,
  :description    => "Count",
  :arguments      => ["--count MESSAGE"],
  :type           => Integer
```

You can force an argument to be boolean:

```ruby
option :detail,
  :description    => "Detailed view",
  :arguments      => ["--detail"],
  :type           => :bool
```

If you have an argument that can be called many times you can force that to build an array:

```ruby
option :args,
  :description    => "Arguments",
  :arguments      => ["--argument ARG"],
  :type           => :array
```

Here if you supplied multiple arguments _configuration[:args]_ will be an array with all the options supplied.

#### Argument validation

You can validate input passed on the CLI:

```ruby
option :count,
  :description    => "Count",
  :arguments      => ["--count MESSAGE"],
  :type           => Integer,
  :validate       => Proc.new {|val| val < 10 ? true : "The message count has to be below 10" }
```

Should the supplied value be 10 or more an error message will be displayed.

#### Disabling standard sections of arguments

By default every Application gets all the RPC options enabling filtering, discovery etc.  In some cases this is undesirable so we let users disable those.

```ruby
class MCollective::Application::Echo<MCollective::Application
   exclude_argument_sections "common", "filter", "rpc"
end
```

This application will only have --help, --verbose and --config as options, all the other options will be removed.

#### Post argument parsing hook

Right after all arguments are parsed you can have a hook in your program called, this hook could perhaps parse the remaining data on _ARGV_ after option parsing is complete.

```ruby
class MCollective::Application::Echo<MCollective::Application
   description "Reports on usage for a specific fact"

   def post_option_parser(configuration)
      unless ARGV.empty?
         configuration[:message] = ARGV.shift
      else
         STDERR.puts "Please specify a message on the command line"
         exit 1
      end
   end

   def main
      # use configuration[:message] here to access the message
   end
end
```

#### Validating configuration

After the options are parsed and the post hook is called you can validate the contents of the configuration:

```ruby
class MCollective::Application::Echo<MCollective::Application
   description "Reports on usage for a specific fact"

   # parse the first argument as a message
   def post_option_parser(configuration)
      configuration[:message] = ARGV.shift unless ARGV.empty?
   end

   # stop the application if we didnt receive a message
   def validate_configuration(configuration)
      raise "Need to supply a message on the command line" unless configuration.include?(:message)
   end

   def main
      # use configuration[:message] here to access the message
   end
end
```

### Exiting your application

You can use the normal _exit_ Ruby method at any time to exit your application and you can supply any
exit code as normal.

The supplied applications have a standard exit code convention, if you want your applications to exhibit
the same behavior use the _halt_ helper.  The exit codes are below:

|Code|Description                                          |
|----|-----------------------------------------------------|
|0   |Nodes were discovered and all passed                 |
|0   |No discovery was done but responses were received    |
|1   |No nodes were discovered                             |
|2   |Nodes were discovered but some responses failed      |
|3   |Nodes were discovered but no responses were received |
|4   |No discovery were done and no responses were received|

```ruby
class MCollective::Application::Echo<MCollective::Application
   description "Reports on usage for a specific fact"

   def main
      mc = rpcclient("echo")

      printrpc mc.echo(:msg => "Hello World", :options => options)

      printrpcstats

      halt mc.stats
   end
end
```

As you can see you pass the _halt_ helper an instance of the RPC Client statistics and it will then
use that to do the right exit code.
