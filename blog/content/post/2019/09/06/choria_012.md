---
title: "Choria Server and Broker 0.12.0"
date: 2019-09-06T9:00:00+01:00
tags: ["releases"]
draft: true
---

The next releases will start coming in over the next week or three, we're getting going with quite a major release for the Choria Server and Broker and a few related packages, I'll introduce some of the changes here today.

Choria Release 0.12.0 is available today, you can get it by updating your Hiera data `choria::version`.

<!--more-->

## NATS 2.0

The fine folk at [NATS.io](https://nats.io) did a major release of their NATS Server technology.  We [blogged about this release recently](https://choria.io/blog/post/2019/08/01/nats_20_networking/) where you can find all the details. The Choria Network Broker is based on this release.

The basic overview is that the NATS Server now has strong security allowing for Multi Tenancy in the broker network, it has a number of other improvements like new ways to extend the network outside of your LAN, new security features and improved observability.

While we do not yet have a full fleshed out documented story for how we'll use all of these features in Choria they are all enabled today.  The Choria Server and Go Client supports NATS credentials too via the `plugin.nats.credentials` settings.

This is a huge step forward for our future scalability needs and as always this has been tested extensively on 50 000 nodes.

## Go 1.13

We now compile the releases with Go 1.13, we support `go mod` and as a side effect we cannot support RHEL 5 anymore, those releases are dropped.

## Dependency Visibility

The `go buildinfo` command now shows all the dependencies compiled into the binary including their versions.  Because we use go 1.13 these are verified using their checksum database during builds.

## Vast improvements to the Go client library

The [Go Client](https://godoc.org/github.com/choria-io/mcorpc-agent-provider/mcorpc/client) and the [Go DDL](https://godoc.org/github.com/choria-io/mcorpc-agent-provider/mcorpc/ddl/agent) had huge improvements which makes it now a full featured client capable of doing everything the Ruby one could do including the more meta things driven by the DDL.

 * Supports limiting targets by number, percentage in random or first modes with repeatable selections by giving a seed
 * Supports calculating aggregates and includes `summary`, `average` and `chart` aggregators
 * Added callbacks to communicate with the calling library about discovery, limits etc to facilitate better user feedback
 * Supports parsing `input` in DDLs
 * Supports validating data against the DDL including data types and validators `shellsafe`, `ipv4address`, `ipv6address`, `ipaddress` and arbitrary regex
 * Supports converting a `map[string]string` of inputs into `map[string]interface{}` where every key is validated and values have the correct types based on the DDL.  Meaning JSON encoding will do the right things

We'll soon add a section of documentation detailing how to use all these features, the new `choria req` CLI uses this (see below).

## A new RPC client

There is now an implementation of the much loved/hated `mco rpc` written in pure Go. It is nearly 1:1 feature compatible with the following notable differences:

 * The `-j` flag still produce JSON but the JSON format has changed and become much more useful, now including stats and aggregates
 * DDL declared aggregates are supported
 * Various outputs are slightly changed and now displays valid JSON where sensible
 * Short versions of some options like `--ln` for `--limit-nodes` are gone due to limitations in my CLI framework, few other small CLI flag changes
 * The `--display` flag now supports `none` in addition to past flags
 * `--verbose` will respect the DDL display hints and `--display`

Other than that it's 1:1 compatible, if I missed any command line flags that you use in the old one please contact me.

![faaaast](rpcclient.gif)

Other than that, it's incredibly fast above you can see a old `mco rpc` vs this new one on the same nodes in a real time comparison, wow.  The ruby `mco rpc` would do a directed request of `rpcutil#ping` in about 30 seconds on 40k nodes.  The `choria req` does it in 2 seconds.

As it's compiled into the choria `binary` it has no external requirements.

I mentioned the `chart` aggregate, as with other aggregate plugins it gives you a birds eye view of your results, it's probably quite a niche one but here it is showing me my total resource counts across my fleet:

```nohighlight
Summary of Total Resources:

  730  ┤                  ╭─╮
  693  ┤                  │ │
  656  ┤                  │ │
  620  ┼╮                ╭╯ │
  583  ┤╰──╮             │  │
  547  ┤   │             │  ╰╮       ╭╮           ╭─────────────╮
  510  ┤   │     ╭╮     ╭╯   │      ╭╯╰╮         ╭╯             ╰╮
  473  ┤   ╰╮    ││     │    │      │  ╰╮       ╭╯               ╰╮
  437  ┤    │   ╭╯│     │    │      │   ╰─────╮╭╯                 ╰
  400  ┤    │   │ ╰╮   ╭╯    │     ╭╯         ╰╯
  363  ┤    ╰╮  │  │   │     ╰╮    │
  327  ┤     │ ╭╯  ╰╮ ╭╯      │    │
  290  ┤     │ │    │ │       │   ╭╯
  253  ┤     ╰╮│    │╭╯       │   │
  217  ┤      ╰╯    ╰╯        │   │
  180  ┤                      ╰───╯
```

Immediately I have a idea that I have some nodes with really very low resource counts (I should probably investigate that can't be right), and some high ones, I do not know which ones or how many but I can dig into this:

```nohighlight
$ choria rpc puppet last_run_summary -j | \
       jq -r '.replies|.[]|select(.data.total_resources < 200)|.sender'
dev10.example.net
dev11.example.net
dev12.example.net
```

By using the new easier to use JSON output and `jq` I could find out what were the weird nodes.

I do wish to revisit the default RPC client and make it a much more user friendly experience, unfortunately I think my Go tooling is not quite up to the task yet.  For now this gets us going.

The `choria ping --graph` output is now using the same graph widget rather than the old tiny sparkline.

## pkcs11 Support

Thanks to the tireless efforts of Paul Tittle the security system now supports pkcs11.  This means you can authenticate to the network using your Yubikey and other HSM. At present this is only available in the Golang client libraries (and so also the new rpc client).

There's a fantastic blog introducing this feature written by Paul: [New pkcs11 Security Provider](../pkcs11).

## Agent Activation changes

We're slowly moving to a world where all agents and clients will be disabled unless specifically enabled. Today we support checking for these configuration entries but we default to `true`, in time this will change to `false`.

The main reason for this is to support a mode of deployment where all our Ruby agent plugins will go into the `choria-mcorpc-support` gem and installing it will bring all the core agents.

This will remove several 100 files from the Puppet File Server and make many people very happy.  To facilitate that we will deactivate all agents and activate them as soon as their configuration classes are included.
