---
title: "Introducing App Builder"
date: 2022-05-25T00:00:00+01:00
tags: ["releases"]
draft: false
---

Today I am pleased to announce a new operations tool, it's a small little thing, but it can be a huge help, I hope you
will love it.

Operations teams tend to use a large selection of shell scripts, piped commands, kubectl invocations and more in their 
day to day job.

To a large extend these are tribal knowledge and something that is a big hurdle for new members of the team. The answer 
is often to write wiki pages capturing run books that has these commands documented.

This does not scale well and does not stay up to date.

What if there was a CLI tool that encapsulated all of these commands in a single, easy to use and easy to discover command.

The [appbuilder](https://github.com/choria-io/appbuilder) project lets you build exactly that by specifying a model for 
your CLI application in a YAML file and then building custom interfaces on the fly.

There will be 2 versions of this, the standalone version mentioned above that has only non Choria related features, we will
then use it as a library in the normal Choria release where it will gain `discovery`, `rpc` and `kv` support.  You will see
that in the next Choria release (0.26.0).

There is an optional [video introducing the idea behind it](https://youtu.be/wbu3N63WY7Y).

See the full entry for an example of this tool in use and what it is all about.

View the [Documentation](https://choria-io.github.io/appbuilder/) and [GitHub Repo](https://github.com/choria-io/appbuilder) for more

<!--more-->

## Example

A picture is worth a thousand words, so this is how it looks in use:

```
$ natsctl --help
usage: natsctl [<flags>] <command> [<args> ...]

NATS Stress Test Cluster Controller

Contact: R.I.Pienaar <rip@devco.net>

Flags:
  --help  Show context-sensitive help (also try --help-long and --help-man).

Commands:
  help [<command>...]
    Show help.

  report servers
    Shows list of NATS servers

  report jetstream
    Shows list of NATS servers

  service list [<flags>]
    List the servers running the service

  service state [<flags>]
    Obtain the state of the service operator
...
```

In the example above one can run commands like `natsctl report servers` or `natsctl report jetstream` these will invoke something like `nats server list --user system --password secret --server nats.example.net:4222`.

The `natscl service state` command invokes a Choria RPC API in a subset of fleet nodes and pass the result through a JQ query `.replies | .[] | select(.statuscode==0) | .sender + ": " + .data.state` to transform the API output.

It is much nicer to just wrap it in `natsctl report servers` or `natsctl service state` and be able to manage the detail seperately.  You can mention `natsctl report servers` in wikis and if it ever changes, you only change the app model.

These sub commands all have help and integration with bash and zsh is provided.


