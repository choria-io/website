+++
title = "About Choria"
weight = 1
icon = "<b>1. </b>"
+++

Choria is an ongoing effort to produce a modern orchestration system that is easy to use and highly scalable.

It provides a compatibility layer for The Marionette Collective plugins safe guarding the investment users made in creating those automations.

## Overview

Choria is an ongoing project to create a new Orchestration System with roots in The Marionette Collective (mcollective) but with a view on the future.  It's part trivially installable mcollective and part brand new development.

Using Choria a Open Source Puppet user can be up and running with a scalable, clustered, secure orchestration system within 30 minutes.  Out of the box it's known to support 50 000 nodes on a single server.

The system will be secure by default, easy to maintain and production ready.

As Puppet Inc is in the process of sunsetting mcollective Choria safe guards the investment users have made through a compatibility framework capable of running mcollective agents.

## Features

  * A complete Playbook system that integrates Choria with any number of other services to conduct full workflow based actions. Playbooks are written using the Puppet DSL.
  * Support for executing Puppet Tasks as found on the Puppet Forge without the need for SSH and with RBAC
  * Integration with the Puppet Certificate Authority system
  * A Discovery plugin for PuppetDB giving you a responsive and predictable interaction mode with support for PQL based infrastructure discovery
  * A Connector middleware technology that has been tested to 100s of thousands of nodes. 50 000 nodes can comfortably be run on a single Choria Broker.
  * Federations of Collectives eases the admin burden of managing large geographically distributed Collectives
  * Full end to end Authentication, Authorization and Auditing out of the box
  * Common Puppet eco system plugins like Package, Service, Puppet and File Manager deployed and ready to use
  * Support for SRV records and sane configuration defaults to attain a zero config setup

See the [Deployment Guide](../deployment) for details on installation.

## Status

This system is production ready but under active development.  At various fronts we are working to replace reliance on Puppet Agent and legacy MCollective, the project lives on [GitHub](https://github.com/choria-io).

Extensive performance testing has been done that showed the system to be stable at over 100 000 nodes.  Getting to 50 000 nodes is easily achievable using a single middleware server.
