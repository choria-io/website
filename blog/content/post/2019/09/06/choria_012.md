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

## A new RPC client

There is now an implementation of the much loved/hated `mco rpc` written in pure Go. It is nearly 1:1 feature compatible with the following notable differences:

 * The `-j` flag still produce JSON but the JSON format has changed and become much more useful, now including stats and aggregates
 * DDL decalred aggregates are support but only `summary`, `boolean_summary` and `average`. These plugins were written in Ruby and cannot be called from Go. The above 3 should cover 90% of real world use.
 * Various outputs are slightly changed and now displays valid JSON where sensible

Other than that it's 1:1 compatible, if I missed any command line flags that you use in the old one please contact me.

Other than that, it's incredibly fast.  The ruby `mco rpc` would do a directed request of `rpcutil#ping` in about 30 seconds on 40k nodes.  The `choria req` does it in 2 seconds.

As it's compiled into the choria `binary` it has no external requirements.

## pkcs11 Support

Thanks to the tireless efforts of Paul Tittle the security system now supports pkcs11.  This means you can authenticate to the network using your Yubikey and other HSM.

There's a fantastic blog introducing this feature: **TODO**

## Agent Activation changes

We're slowly moving to a world where all agents and clients will be disabled unless specifically enabled. Today we support checking for these configuration entries but we default to `true`, in time this will change to `false`.

The main reason for this is to support a mode of deployment where all our Ruby agent plugins will go into the `choria-mcorpc-support` gem and installing it will bring all the core agents.

This will remove several 100 files from the Puppet File Server and make many people very happy.  To facilitate that we will deactivate all agents and activate them as soon as their configuration classes are included.