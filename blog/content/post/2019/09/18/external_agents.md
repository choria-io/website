---
title: "Writing agents in any language"
date: 2019-09-18T9:00:00+01:00
tags: ["agents", "extending", "developing"]
draft: false
---

MCollective has for a long time been extendible using purely Ruby means. This was fine in an earlier age where it seemed Ruby was going to rule the ops space but turned out that isn't how things ended up, and even then there were lots of real interest in extending it using Python for example.

In the last 2 weeks this conversation came up again and in many respects the situation was worse now since the new Choria Server being written in Go is not extendible by external plugins, we supported the old Ruby ones but that was it. We had plans to support [Tengo](https://github.com/d5/tengo) and [Yaegi](https://blog.containo.us/announcing-yaegi) as ways to add agents to Choria but neither of those got past POC.

There were major hurdles to actually doing this in the new Go system:

 * no `action_policy` type authorization system
 * no `data` plugins
 * no data aggregators
 * limited DDLs
 * do not have inputs and outputs properly mapped
 * have no DDL validators
 * have no way to set default data in requests

These missing features all worked fine for the ruby agents though - since the MCollective compatability layer would just start up a subset of old MCollective on demand and so gain access to its versions of all of the above features.  We did not need any of it.

In the last 3 weeks we addressed most of these missing features in the pure go daemon:

 * We have `shellsafe`, `ipv4address`, `ipv6address`, `ipaddress` and `regex` validators. Analysis showed this hits 90%+ of what was ever used
 * We have data aggregators for `average`, `summary` (including booleans) and a new `chart` one. Analysis showed these would hit 90%+ of what was ever used
 * We have greatly extended the DDLs with full `input`, `output` and `aggregate` awareness
 * We can set defaults in both `inputs` and `outputs` and it's all type aware
 * We can do full validation of requests based on the DDLs
 * We have a `action_policy` plugin thats 1:1 compatible except for compound statements (planned)
 * We can generate ruby DDLs from JSON ones

This has been a huge push in features. So at this point if we add new ways to write agents they would get all these features for free and suddenly the prospect of doing just that is a lot more palatable. But why stop at supporting a specific language like `tengo`? Why not support all languages - especially with new movement in things like `webasm`?

That's exactly what I did in a new feature called `External Agents` and it will be available in the next release, read on for the full details.

<!--more-->

## External Agents Overview

External agents are executables that receive a request in a temporary file in JSON format and can reply by writing JSON to a reply file. That's it.

These external agents live in the same directories as other agents, can be packaged using `mco plugin package` just like any other, distributed on the Forge and for all intends and purposes they act exactly like any other agent including being compatible with PuppetDB based discovery.  You can interact with them using the same `mco` CLI tools, APIs and so forth.

They are subject to all the DDL controls - full validation and input/output defaults. They are subject to full AAA in a 100% compatible manner with any you already know.

Your code will never be called for requests that fail DDL validation or AAA controls.

They integrate with the Choria Server logging - any output on `STDOUT` is logged at `INFO` level while `STDERR` output is logged `ERROR` level.

It supports an activation check allowing your agent to ensure that all dependencies it needs is available - or indeed if this is an appropriate machine to expose a given feature.

## Example

Lets make a simple `echo` agent that can echo back a message sent to it.

Our agent will respond to a single action `ping` which takes a string input `message`. It will reply with the same string `message` and a integer `timestamp`.

Now we'll create a simple agent in Ruby, here I am not using any helpers or library to make it easier, we anticipate the community will provide a few and indeed a [Python one](https://github.com/optiz0r/py-mco-agent) is already being written by the community.

```ruby
#!/usr/bin/ruby

require "json"

# parse the incoming request
req = JSON.parse(File.read(ENV["CHORIA_EXTERNAL_REQUEST"]))

# respond depending on protocol type
case ENV["CHORIA_EXTERNAL_PROTOCOL"]

# we always activate
when "choria:mcorpc:external_activation_check:1"
  File.open(ENV["CHORIA_EXTERNAL_REPLY"], "w") {|f| f.print JSON.dump("activate" => true)}

# this is our reply as described
when "choria:mcorpc:external_request:1"
  rep = {
    "statuscode" => 0,
    "statusmsg" => "OK",
    "data" => {
        "message" => req["body"]["data"]["message"],
        "timestamp" => Time.now.utc.to_i
    }
  }

  File.open(ENV["CHORIA_EXTERNAL_REPLY"], "w") {|f| f.print JSON.dump(rep)}

else
  raise("unknown protocol: `%s`" % ENV["CHORIA_EXTERNAL_PROTOCOL"])
end
```

That's all that's needed, in reality we'd add more structure, some error handling etc - but those would be handled by the above mentioned helper libraries.

You'll need a DDL, generate it using `choria tool generate ddl echo.json echo.ddl`.

You'd stick these in a work directory `echo/agent`:

```nohighlight
-rwxrwxr-x 1 rip rip 1185 Sep 16 13:11 echo
-rw-rw-r-- 1 rip rip  810 Sep 18 12:37 echo.ddl
-rw-rw-r-- 1 rip rip 1048 Sep 18 12:37 echo.json
```

Then package it using `mco plugin package`:

```nohighlight
$ mco plugin package --format aiomodulepackage --vendor ripienaar
Completed building module for mcollective_agent_echo
$ ls -l ripienaar-mcollective_agent_echo-1.0.0.tar.gz
```

Once installed using the usual method, it's accessible as here:

```nohighlight
$ choria req helloworld ping message=hello
Discovering nodes .... 1

1 / 1    0s [====================================================================] 100%

example.net
     Message: hello
   Timestamp: 1568803451


Finished processing 1 / 1 hosts in 507.08144ms
```

## Status

As I mentioned these feature will be in the next release which will come out soon (nightly builds already have it), we have some testing and validation to do and specifically I will not enable the new `action_policy` initially by default while things settle and any bugs get squashed.

Full documentation for the feature can be found on our website: [External Agents](https://choria.io/docs/development/mcorpc/externalagents/).

I am excited about the capability this brings and hopefully it will help people get going with higher order management using their own tech choices and not ones the Choria project prescribe.