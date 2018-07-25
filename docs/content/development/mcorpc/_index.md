+++
title = "MCollective RPC"
+++

MCollective RPC is a framework that first appeared in The Marionette Collective that Choria provides a compatability layer for.

It provides the following:

* Conventions for writing [agents](agents/) and [clients](clients/), favoring convention over custom design
* Easy to write agents including input validation and a sensible feedback mechanism in case of error
* Provide audit logs of all calls to agents
* Provide the ability to do fine grain Authorization of calls to agents and actions.
* Has a Data Definition Language used to describe agents and assist in giving hints to auto generating user interfaces.
* The provided generic client tool - `mco rpc` - should be able to speak to most compliant agents
* Should you need to you can still write your own clients, this should be very easy too
* Return data should be easy to print, in most cases the framework should be able to print a sensible output with a single, provided, function.  The DDL is used here to improve the standard one-size-fits-all methods.
* [Standardised packaging](/docs/configuration/plugins/) on the Puppet Forge using Puppet Modules.

We've provided full tutorials on [Writing RPC Clients](clients/) and [Agents](agents/). Numerous full featured examples can be found in our [GitHub Project dedicated to plugins](https://github.com/choria-plugins).


A bit of code probably says more than lots of English, so here's a simple hello world Agent, it just echo's back everything you send it in the _:msg_ argument:

```ruby
module MCollective
  module Agent
    class Helloworld<RPC::Agent
      # Basic echo server
      action "echo" do
        validate :msg, String

        reply.data = request[:msg]
      end
    end
  end
end
```

The nice thing about using a standard abstraction for clients is that you often won't even need to write a client for it, we ship a standard client that you can use to call the agent above:

 ```nohighlight
 % mco rpc helloworld echo msg="Welcome to MCollective RPC"
 Determining the amount of hosts matching filter for 2 seconds .... 1

 devel.your.com                          : OK
     "Welcome to MCollective RPC"



 ---- rpctest#echo call stats ----
            Nodes: 1
       Start Time: Wed Dec 23 20:49:14 +0000 2009
   Discovery Time: 0.00ms
       Agent Time: 54.35ms
       Total Time: 54.35ms
```

Note: Please see the [agents](agents/) and [clients](clients/) pages for a thorough walkthrough.

But you can still write your own clients, it's incredibly simple, full details of a client is out of scope for the introduction - see the [Writing Clients](clients/) page instead for full details - but here is some sample code to do the same call as above including full discovery and help output:

```ruby
#!/bin/env ruby

require "mcollective"

include MCollective::RPC

mc = rpcclient("helloworld")

printrpc mc.echo(:msg => "Welcome to MCollective RPC")

printrpcstats
```

The above code effectively achieves the same as the command `mco rpc helloworld echo msg="Welcome to MCollective RPC"` and like that command you can pass `--help` to your script and do full filtering of which nodes to communicate with and more.
