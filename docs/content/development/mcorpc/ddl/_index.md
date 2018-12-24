+++
title = "DDL Files"
weight = 30
+++

As with other remote procedure invocation systems MCollective has a DDL that defines what remote methods are available, what inputs they take and what outputs they generate.

In addition to the usual procedure definitions we also keep meta data about author, versions, license and other key data points.

The DDL is used in various scenarios:

* The user can access it in the form of a human readable help page
* User interfaces can access it in a way that facilitate auto generation of user interfaces
* The RPC client auto configures and use appropriate timeouts in waiting for responses
* Before sending a call over the network inputs get validated so we do not send unexpected data to remote nodes.
* Module repositories can use the meta data to display a standard view of available modules to assist a user in picking the right ones.
* The server will validate incoming requests prior to sending it to agents

{{% notice warning %}}
This section of the documentation have recently been migrated from the old mcollective documentation, we are still in the process of verifying every example works in modern mcollective.  If you find any issues please get in touch.
{{% /notice %}}

## Examples

We'll start with a few examples as I think it's pretty simple what they do, and later on show what other permutations are allowed for defining inputs and outputs.

The typical service agent is a good example, it has various actions that all more or less take the same input.  All but status would have almost identical language.

### Meta Data

First we need to define the meta data for the agent itself:

```ruby
metadata :name        => "service",
         :description => "Agent to manage services using the Puppet service provider",
         :author      => "R.I.Pienaar",
         :license     => "GPLv2",
         :version     => "1.1",
         :url         => "https://docs.puppetlabs.com/mcollective/plugin_directory/",
         :timeout     => 60
```

It's fairly obvious what these all do, _:timeout_ is how long the MCollective daemon will let the threads run.

## Required versions

You can indicate which is the lowest version of MCollective needed to use a plugin.  Plugins that do not meet the requirement can not be used.

```ruby
requires :mcollective => "2.0.0"
```

You should add this right after the metadata section in the DDL

## Actions, Input and Output

Defining inputs and outputs is the hardest part, below first the _status_ action:

```ruby
action "status", :description => "Gets the status of a service" do
    display :always  # supported in 0.4.7 and newer only

    input :service,
          :prompt      => "Service Name",
          :description => "The service to get the status for",
          :type        => :string,
          :validation  => '^[a-zA-Z\-_\d]+$',
          :optional    => false,
          :default     => "mcollective",
          :maxlength   => 30

    output :status,
           :description => "The status of service",
           :display_as  => "Service Status",
           :default     => "unknown status"
end
```

As you see we can define all the major components of input and output parameters.  _:type_ can be one of various values and each will have different parameters, more on that later.

For agents the reply structures are pre-populated with all the defined outputs, if no default is supplied a default of nil will be set.

By default mcollective only show data from actions that failed, the _display_ line above tells it to always show the results.  Possible values are _:ok_, _:failed_ (the default behavior) and _:always_.

Finally the service agent has 3 almost identical actions - _start_, _stop_ and _restart_ - below we use a simple loop to define them all in one go.

```ruby
["start", "stop", "restart"].each do |act|
    action act, :description => "#{act.capitalize} a service" do
        input :service,
              :prompt      => "Service Name",
              :description => "The service to #{act}",
              :type        => :string,
              :validation  => '^[a-zA-Z\-_\d]+$',
              :optional    => false,
              :maxlength   => 30

        output :status,
               :description => "The status of service after #{act}",
               :display_as  => "Service Status",
               :default     => "unknown status"
    end
end
```

All of this code just goes into a file, no special class or module bits needed, just save it as _service.ddl_ in the same location as the _service.rb_.

Importantly you do not need to have the _service.rb_ on a machine to use the DDL, this means on machines that are just used for running client programs you can just drop the _.ddl_ files into the agents directory.

You can view a human readable version of this using *mco plugin doc &lt;agent&gt;* command:

```nohighlight
% mco plugin doc service
service
=======

Agent to manage services using the Puppet service provider

      Author: R.I.Pienaar
     Version: 1.1
     License: GPLv2
     Timeout: 60
   Home Page: https://docs.puppetlabs.com/mcollective/plugin_directory/



ACTIONS:
========
   restart, start, status, stop

   restart action:
   ---------------
       Restart a service

       INPUT:
           service:
              Description: The service to restart
                   Prompt: Service Name
                     Type: string
               Validation: ^[a-zA-Z\-_\d]+$
                   Length: 30


       OUTPUT:
           status:
              Description: The status of service after restart
               Display As: Service Status
```

### Optional Inputs

The input block has a mandatory _:optional_ field, when true it would be ok if a client attempts to call the agent without this input supplied.  If it is supplied though it will be validated.

### Types of Input

As you see above the input block has _:type_ option, types can be _:string_, _:list_, _:boolean_, _:integer_, _:float_ or _:number_

#### :string type

The string type validates initially that the input is in fact a String, then it validates the length of the input and finally matches the supplied Regular Expression.

Both _:validation_ and _:maxlength_ are required arguments for the string type of input.

If you want to allow unlimited length text you can make _:maxlength => 0_ but use this with care.

A plugin type called [Validator Plugins](../vaildator/) exist that allow you to supply your own validations for *:string* types.

#### :list type

List types provide a list of valid options and only those will be allowed, see an example below:

```ruby
input :action,
      :prompt      => "Service Action",
      :description => "The action to perform",
      :type        => :list,
      :optional    => false,
      :list        => ["stop", "start", "restart"]
```

In user interfaces this might be displayed as a drop down list selector or another kind of menu.

#### :boolean type

The value input should be either _true_ or _false_ actual boolean values.

#### :integer type

The value input should be an integer number like _1_ or _100_ but not _1.1_.

#### :float type

The value input should be a floating point number like _1.0_ but not _1_.

#### :number type

The value input should be an integer or a floating point number.

### Accessing the DDL

While programming client applications or web apps you can gain access to the DDL for any agent in several ways:

```ruby
require 'mcollective'

config = MCollective::Config.instance
config.loadconfig(options[:config])

ddl = MCollective::DDL.new("service")
puts ddl.help("#{config.configdir}/rpc-help.erb")
```

This will produce the text help output from the above example, you can supply any ERB template to format the output however you want.

You can also access the data structures directly:

```ruby
ddl = MCollective::DDL.new("service")
puts "Meta Data:"
pp ddl.meta

puts
puts "Status Action:"
pp ddl.action_interface("status")
```

```nohighlight
Meta Data:
{:license=>"GPLv2",
 :author=>"R.I.Pienaar",
 :name=>"service",
 :timeout=>60,
 :version=>"1.1",
 :url=>"https://docs.puppetlabs.com/mcollective/plugin_directory/",
 :description=>"Agent to manage services using the Puppet service provider"}

Status Action:
{:action=>"status",
 :input=>
  {:service=>
    {:validation=>"^[a-zA-Z\\-_\\d]+$",
     :maxlength=>30,
     :prompt=>"Service Name",
     :type=>:string,
     :optional=>false,
     :description=>"The service to get the status for"}},
 :output=>
  {"status"=>
    {:display_as=>"Service Status", :description=>"The status of service"}},
 :description=>"Gets the status of a service"}
```

The ddl object is also available on any *rpcclient*:

```ruby
service = rpcclient("service")
pp service.ddl.meta
```

In the case of accessing it through the service as in this example, if there was no DDL file on the machine for the service agent you'd get a *nil* back from the ddl accessor.

### Input Validation

As mentioned earlier the client does automatic input validation using the DDL, if validation fails you will get an *MCollective::DDLValidationError* exception thrown with an appropriate message.
