+++
title = "Puppet Tasks"
icon = "<b>5. </b>"
+++

## Introduction

Puppet Tasks are new containers for commands, much like MCollective Agents they have metadata that can describe their inputs and outputs and they are distrubted using the Puppet Forge.

These can be much easier to write than MCollective Agents as their metadata is very light weight and can progressively become more complex as your needs grow.  Tasks can be written using any Programming Language.

Tasks are distributed using the Puppet Forge and one can search for [all modules with tasks](https://forge.puppet.com/modules?utf-8=%E2%9C%93&sort=rank&q=&endorsements=&with_tasks=yes).

Choria provides support for tasks as a new first class agent type, accessible over the normal RPC api and provides a rich CLI.

## Benefits

Choria provides support for Puppet Tasks to the Open Source Community.  Choria provides a strong enterprise focussed workflow with the following features:

  * Tasks are downloaded from the Puppet Server to the nodes. You have a single point of code management and it's not 100s of admins home directory
  * Does not need SSH or direct access to nodes
  * Individual Tasks are subject to Role Based Access Control using the standard Action Policy
  * Individual Task invocations are audited using the standard Choria Auditing
  * Tasks run in the background and are suitable for long running actions like system updates
  * A _mco tasks_ command that behaves like a typical mcollective utility
  * Tasks can be used from your ruby code or playbooks using the _bolt\_tasks_ agent

## Requirements

  * Your client and all your nodes must run Puppet 5.4.0 at least.
  * You must have a version 5.1 or newer Puppet Server where your code is hosted, tasks are downloaded from here
  * You must have Choria 0.7.0 or newer

## Demonstration

The pages in this section will provide a thorough reference for this feature, you can also watch a video showcasing the capabilities of Puppet Tasks in Choria

<iframe width="560" height="315" src="https://www.youtube.com/embed/LLyjPjZW7TE" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

## Status

At present I consider this feature a preview quality feature.  You can follow [open issues in GitHub](https://github.com/choria-io/mcollective-choria/issues?q=is%3Aissue+is%3Aopen+label%3Atasks) to get an idea of what is out standing.
