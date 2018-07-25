+++
title = "Clients"
weight = 10
+++

As pointed out in the Introduction page you can use the _mco rpc_ CLI to call agents and it will do it's best to print results in a sane way.  When
this is not enough you can write your own clients.

This guide shows you how to use the Client API to interact with MCollective RPC agents.  This API can be used as here in standalone scripts but you can also apply the knowledge documented here to create [new sub commands](../applications/) like _mco yourapp_ for when you wish to have entirely custom UIs above those that the one-size-fits-all _mco rpc_ provides.

We'll walk through building a ever more complex example of Hello World here that you saw in the introduction section.

{{% notice warning %}}
This section of the documentation have recently been migrated from the old mcollective documentation, we are still in the process of verifying every example works in modern mcollective.  If you find any issues please get in touch.
{{% /notice %}}

## The Basic Client

The client consists of helper methods that you use as a [Ruby Mixin][RubyMixin] in your own code, it provides:

* Standard command line option parsing with help output
* Ability to add your own command line options
* Access to agents and actions
* Tools to help you print results
* Tools to print stats
* Tools to construct your own filters
* Behaviour consistent with the standard [CLI Interaction Model](/concepts/cli/)

We'll write a client for the _Helloworld_ agent that you saw in the Introduction section.

## Call an Agent and print the result

A basic hello world client can be seen below:

```ruby
#!/opt/puppetlabs/puppet/bin/ruby

require "mcollective"

include MCollective::RPC

mc = rpcclient("helloworld")

printrpc mc.echo(:msg => "Welcome to MCollective RPC")

printrpcstats
```

Save this into _hello.rb_ and run it with _--help_, you should see the standard basic help including filters for discovery.

If you've set up the Agent and run the client you should see something along these lines:

```nohighlight
$ hello.rb

Finished processing 44 hosts in 375.57 ms
```

While it ran you would have seen a little progress bar and then just the summary line.  The idea is that if you're talking to a 1000 machine there's no point in seeing a thousand _OK_, you only want to see what failed and this is exactly what happens here, you're only seeing errors and since there were none, no results are shown.

If you run it with _--verbose_ you'll see a line of text for every host and also a larger summary of results.  Running with _--display all_ will show all results with a normal summary.

I'll explain each major line in the code below then add some more features from there:

```ruby
include MCollective::RPC

mc = rpcclient("helloworld")
```

The first line pulls in the various helper functions that we provide, this is the Mixin we mentioned earlier.

We then create a new client to the agent _helloworld_ that you access through the _mc_ variable.

```ruby
printrpc mc.echo(:msg => "Welcome to MCollective RPC")

printrpcstats
```

To call a specific action you have to do _mc.echo_ this calls the _echo_ action, we pass a _:msg_ parameter into it with the string we want echo'd back.  The parameters will differ from action to action.  It returns a array of the results that you can print any way you want, we'll show that later.

_printrpc_ and _printrpcstats_ are functions used to print the results and stats respectively.

## Adjusting the output

### Verbosely displaying results

As you see there's no indication that discovery is happening and as pointed out we do not display results that are ok, you can force verbosity as below on individual requests:

```ruby
mc = rpcclient("helloworld")

mc.discover :verbose => true

printrpc(mc.echo(:msg => "Welcome to MCollective RPC"), :verbose => true)
```

Here we've added a _:verbose_ flag and we've specifically called the discover method.  Usually you don't need to call discover it will do it on demand.  Doing it this way you'll always see the line:

```ruby
Determining the amount of hosts matching filter for 2 seconds .... 44
```

Passing verbose to _printrpc_ forces it to print all the results, failures or not.

If you just wanted to force verbose on for all client interactions, do:

```ruby
mc = rpcclient("helloworld")
mc.verbose = true

printrpc mc.echo(:msg => "Welcome to MCollective RPC")
```

In this case everything will be verbose, regardless of command line options.

### Disabling the progress indicator

You can disable the twirling progress indicator easily:

```ruby
mc = rpcclient("helloworld")
mc.progress = false
```

Now whenever you call an action you will not see the progress indicator.

### Saving the reports in variables without printing

You can retrieve the stats from the clients and also get text of reports without printing them:

```ruby
stats = mc.echo(:msg => "Welcome to MCollective RPC").stats

report = stats.report
```

_report_ will now have the text that would have been displayed by 'printrpcstats' you can also use _no\_response\_report_ to get report text for just the list of hosts that didnt respond.

If you didn't want to just print the results out to STDOUT you can also get them back as just text:

```ruby
report = rpcresults(mc.echo(:msg => "Welcome to MCollective RPC"))
```

## Applying filters programatically

You can pass filters on the command line using the normal _--with-*_ options but you can also do it programatically.  Here's a new version of the client that only calls machines with the configuration management class _/dev\_server/_ and the fact _country=uk_

```
mc = rpcclient("helloworld")

mc.class_filter /dev_server/
mc.fact_filter "country", "uk"

printrpc mc.echo(:msg => "Welcome to MCollective RPC")
```

You can set other filters like _agent\_filter_, _identity\_filter_ and _compound\_filter_.

The _fact\_filter_ method supports a few other forms in adition to above:

```ruby
mc.fact_filter "country=uk"
mc.fact_filter "physicalprocessorcount", "4", ">="
```

This will limit it to all machines in the UK with more than 3 processors.

## Resetting filters to empty

If while using the client you wish to reset the filters to an empty set of filters - containing only the agent name that you're busy addressing you can do it as follows:

```ruby
mc = rpcclient("helloworld")

mc.class_filter /dev_server/

mc.reset_filter
```

After this code snippet the filter will only have an agent filter of _helloworld_ set.

## Processing Agents in Batches

By default the client will communicate with all machines at the same time.
This might not be desired as you might affect a DOS on related components.

You can instruct the client to communicate with remote agents in batches
and sleep between each batch.

Any client application has this capability using the _--batch_ and _--batch-sleep-time_
command line options.

You can also enable this programatically either per client or per request:

```ruby
mc = rpcclient("helloworld")
mc.batch_size = 10
mc.batch_sleep_time = 5

mc.echo(:msg => "hello world")
```

By default batching is disabled and sleep time is 1

Setting the _batch\_size_ to 0 will disable batch mode in both examples above,
effectively overriding what was supplied on the command line.

## Forcing Rediscovery

By default it will only do discovery once per script and then re-use the results, you can though force rediscovery if you had to adjust filters mid run for example.

```ruby
mc = rpcclient("helloworld")

mc.class_filter /dev_server/
printrpc mc.echo(:msg => "Welcome to MCollective RPC")

mc.reset

mc.fact_filter "country", "uk"
printrpc mc.echo(:msg => "Welcome to MCollective RPC")
```

Here we make one _echo_ call - which would do a discovery - we then reset the client, adjust filters and call it again.  The 2nd call would do a new discovery and have new client lists etc.

## Supplying your own discovery information

A new core messaging mode has been introduced that enables direct non filtered communicatin to specific nodes.  This has enabled us to provide an discovery-optional
mode but only if the collective is configured to support direct messaging.

```ruby
mc = rpcclient("helloworld")

mc.discover(:nodes => ["host1", "host2", "host3"]

printrpc mc.echo(:msg => "Welcome to MCollective RPC")
```

This will immediately, without doing discovery, communicate just with these 3 hosts.  It will do normal failure reporting as with normal discovery based
requests but will just be much faster as the 2 second discovery overhead is avoided.

The goal with this feature is for cases such as deployment tools where you have a known expectation of which machines to deploy to and you always want
to know if that fails.  In that use case a discovery based approach is not 100% suitable as you won't know about down machines.  This way you can provide
your own source of truth.

When using the direct mode messages have a TTL associated with them that defaults to 60 seconds.  Since 1.3.2 you can set the TTL globally in the configuration
file but you can also set it on the client:

```ruby
mc = rpcclient("helloworld")
mc.ttl = 3600

mc.discover(:nodes => ["host1", "host2", "host3"]

printrpc mc.echo(:msg => "Welcome to MCollective RPC")
```

With the TTL set to 3600 if any of the hosts are down at the time of the request the request will wait on the middleware and should they come back up
before 3600 has passed since request time they will then perform the requested action.

## Only sending requests to a subset of discovered nodes

By default all nodes that get discovered will get the request.  This isn't always desirable maybe you want to deploy only to a random subset of hosts or maybe you have a service exposed over MCollective that you want to treat as a HA service and so only speak with one host that provides the functionality.

You can limit the hosts to talk to either using a number or a percentage, the code below shows both:

```ruby
mc = rpcclient("helloworld")

mc.limit_targets = "10%"
printrpc mc.echo(:msg => "Welcome to MCollective RPC")
```

This will pick 10% of the discovered hosts - or 1 if 10% is less than 1 - and only target those nodes with your request.  You can also set it to an integer.

There are two possible modes for choosing the targets.  You can configure a global default method but also set it on your client:

```ruby
mc = rpcclient("helloworld")

mc.limit_targets = "10%"
mc.limit_method = :random
printrpc mc.echo(:msg => "Welcome to MCollective RPC")
```

The above code will force a _:random_ selection, you can also set it to _:first_

## Dealing with the results directly

The biggest reason that you'd write custom clients is probably if you wanted to do custom processing of the results, there are 2 options to do it.

### Results and Exceptions

Results have a set structure and depending on how you access the results you will either get Exceptions or result codes.

|Status Code|Description|Exception Class|
|-----------|-----------|---------------|
|0|OK| |
|1|OK, failed.  All the data parsed ok, we have a action matching the request but the requested action could not be completed.|RPCAborted|
|2|Unknown action|UnknownRPCAction|
|3|Missing data|MissingRPCData|
|4|Invalid data|InvalidRPCData|
|5|Other error|UnknownRPCError|

Just note these now, I'll reference them later down.

### MCollective RPC style results

MCollective RPC provides a trimmed down version of results from the basic Client library.  You'd choose to use this if you just want to do basic things or maybe you're just learning Ruby.  You'll get to process the results _after_ the call is either done or timed out completely.

This is an important difference between the two approaches, in one you can parse the results as it comes in, in the other you will only get results after processing is done.  This would be the main driving facter for choosing one over the other.

Here's an example that will print out results in a custom way.

```ruby
mc.echo(:msg => "hello world").each do |resp|
   puts "%-40s: %s" % [resp[:sender], resp[:data][:msg]]
end
```

This will produce a result something like this:

```nohighlight
dev1.example.net                          : hello world
dev2.example.net                          : hello world
dev3.example.net                          : hello world
```

The _each_ in the above code just loops through the array of results.  Results are an array of Hashes, the data you got for above has the following structure:

```ruby
[{:statusmsg=>"OK",
 :sender=>"dev1.example.net",
 :data=>{:msg => "hello world"},
 :statuscode=>0},
{:statusmsg=>"OK",
 :sender=>"dev2.example.net",
 :data=>{:msg => "hello world"},
 :statuscode=>0}]
```

The _:statuscode_ matches the table above so you can make decisions based on each result's status.

### Gaining access to MCollective::Client#req results

You can get access to each result in real time, in this case you will supply a block that will be invoked for each result as it comes in.  The result set will be exactly as from the full blown client.

In this mode there will be no progress indicator, you'll deal with results as and when they come in not after the fact as in the previous example.

```ruby
mc.echo(:msg => "hello world") do |resp|
   if resp[:body][:statuscode] == 0
      puts "%-40s: %s" % [resp[:senderid], resp[:body][:data]]
   else
      puts "The RPC agent returned an error: #{resp[:body][:statusmsg]}"
   end
end
```

The output will be the same as above

In this mode the results you get will be like this:

```ruby
{
 :msgtarget => "/topic/mcollective.helloworld.reply",
 :senderid => "dev2.example.net",
 :msgtime => 1261696663,
 :hash => "2d37daf690c4bcef5b5380b1e0c55f0c",
 :requestid => "2884afb0b52cb38ea4d4a3146d18ef5f",
 :senderagent => "helloworld"
 :body => {
   :statusmsg => "OK",
   :statuscode => 0,
   :data => {
    :msg => "hello world"
   }
 }
}
```

You can additionally gain access to a RPC style result in addition to the more complex native client results:

```ruby
mc.echo(:msg => "hello world") do |resp, sresp|
   if resp[:body][:statuscode] == 0
      puts "%-40s: %s" % [sresp[:sender], sresp[:data][:msg]]
   else
      puts "The RPC agent returned an error: #{resp[:body][:statusmsg]}"
   end
end
```

You can still use printrpc to print these style of results and gain advantage of the DDL and so forth:

```ruby
mc.echo(:msg => "hello world") do |resp, sresp|
   printrpc sresp
end
```

This should give you a result set that is easier to deal with

## Adding custom command line options

You can look at the _mco rpc_ script for a big sample, here I am just adding a _--msg_ option to our script so you can customize the message that will be sent and received.

```ruby
#!/usr/bin/ruby

require "mcollective"

include MCollective::RPC

options = rpcoptions do |parser, options|
   parser.define_head "Generic Echo Client"
   parser.banner = "Usage: hello [options] [filters] --msg MSG"

   parser.on('-m', '--msg MSG', 'Message to pass') do |v|
      options[:msg] = v
   end
end

unless options.include?(:msg)
   puts("You need to specify a message with --msg")
   exit! 1
end

mc = rpcclient("helloworld", :options => options)

mc.echo(:msg => options[:msg]).each do |resp|
   puts "%-40s: %s" % [resp[:sender], resp[:data][:msg]]
end
```

This version of the code should be run like this:

```noghighlight
% test.rb --msg foo
dev1.example.net                          : foo
dev2.example.net                          : foo
dev3.example.net                          : foo
```

Documentation for the Options Parser can be found [in its code][OptionParser].

And finally if you add options as above rather than try to parse it yourself you will get help integration for free:

```nohighlight
% hello.rb --help
Usage: hello [options] [filters] --msg MSG
Generic Echo Client
    -m, --msg MSG                    Message to pass

Common Options
    -c, --config FILE                Load configuratuion from file rather than default
        --dt SECONDS                 Timeout for doing discovery
        --discovery-timeout
    -t, --timeout SECONDS            Timeout for calling remote agents
    -q, --quiet                      Do not be verbose
    -v, --verbose                    Be verbose
    -h, --help                       Display this screen

Host Filters
        --wf, --with-fact fact=val   Match hosts with a certain fact
        --wc, --with-class CLASS     Match hosts with a certain configuration management class
        --wa, --with-agent AGENT     Match hosts with a certain agent
        --wi, --with-identity IDENT  Match hosts with a certain configured identity
```

## Disabling command line parsing and supplying your options programatically

Sometimes, perhaps when embedding an MCollective client into another tool like Puppet, you do not want MCollective to do any command line parsing as there might be conflicting command line options etc.

This can be achieved by supplying an options hash to the RPC client:

```ruby
include MCollective::RPC

options =  MCollective::Util.default_options

client = rpcclient("test", {:options => options})
```

This will create a RPC client for the agent test without any options parsing at all.

To set options like discovery timeout and so forth you will need use either the client utilities or manipulate the hash upfront, the client utility methods is the best.   The code below sets the discovery timeout in a way that does not require you to know any internal structures or the content of the options hash.

```ruby
options =  MCollective::Util.default_options

client = rpcclient("test", {:options => options})
client.discovery_timeout = 4
```

Using this method of creating custom options hashes mean we can make internal changes to MCollective without affecting your code in the future.

## Sending RPC requests without discovery and blocking

Usually this section will not apply to you.  The client libraries support sending a request without waiting for a reply.  This could be useful if you want to clean yum caches but don't really care if it actually happens everywhere.

You will loose these abilities:

 * Knowing if your request was received by any agents
 * Any stats about processing times etc
 * Any information about the success or failure of your request

The above should make it clear already that this is a limited use case, it's essentially a broadcast request with no feedback loop.

The code below will send a request to the _runonce_ action for an agent _puppetd_, once the request is dispatched I will have no idea if it got handled etc, my code will just continue onwards.

```ruby
p = rpcclient("puppetd")

p.identity_filter "your.example.net"
reqid = p.runonce(:forcerun => true, :process_results => false)
```

This will honor any attached filters set either programatically or through the command line, it will send the request but will
just not handle any responses.  All it will do is return the request id.
