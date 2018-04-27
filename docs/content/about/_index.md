+++
title = "About Choria"
pre = "<b>1. </b>"
menu = "about"
weight = 10
+++

## Overview

Choria is an ongoing project to create a new Orchestration System with roots in The Marionette Collective (mcollective) but with a view on the future.  It's part trivially installable mcollective and part brand new development.

Using Choria an Open Source Puppet user can be up and running with a scalable, clustered, secure orchestration system within 30 minutes.  Out of the box it's known to support 50 000 nodes on a single compute node.

The system is secure by default, easy to maintain and production ready. Once invested in deploying this many avenues for using it exist including turn key end user capabilities.

As Puppet Inc is in the process of sunsetting The Marionette Collective the Choria Project safe guards the investment users have made through a compatibility framework capable of running mcollective agents.

## End User Features

  * A complete [Playbook System](/docs/playbooks/) that integrates Choria with any number of other services to conduct full workflow based actions. Playbooks are written using the Puppet DSL.
  * Support for executing [Puppet Tasks](/docs/tasks/) as found on the Puppet Forge without the need for SSH and with RBAC
  * A Discovery plugin for PuppetDB giving you a responsive and predictable interaction mode with support for PQL based infrastructure discovery
  * Common plugins like Package, Service, Puppet and File Manager deployed and ready to use, many more actively maintained and ready to install.

## Developer Features

  * Write your own Agents that expose new Actions unique to your environment and interact with them from Playbooks, CLI and your own custom Clients.
  * [Embed a Choria Server into your own Golang software](https://github.com/ripienaar/embedded-choria-sample#readme) to create a fast and secure management backplane for any software
  * [Embed a Choria Server into your IoT platform](https://github.com/ripienaar/choriapi) to create a secure data access layer and life cycle management
  * Produce a custom build of Choria with auto provisioning, custom security and custom automation agents compiled right into the binary, using your own branding and paths without a reliance on Puppet
  * Create highly scalable data pipe lines with node data being processed through Stream Processors like NATS Streaming using Choria Data Adapters, ideal for IoT or managing very large geo diverse server estates
  * Build vastly scalable asset data bases using Stream Processing techniques and Choria Metadata

## Operations Features

  * Integration with the Puppet Certificate Authority system
  * A middleware technology that is used in production on 100s of thousands of nodes. 50 000 nodes can comfortably be run on a single Choria Broker.
  * Federations of Collectives eases the admin burden of managing large geographically distributed Collectives
  * Full end to end Authentication, Authorization and Auditing out of the box
  * Support for SRV records and sane configuration defaults to attain a zero config setup

See the [Deployment Guide](../deployment) for details on installation.

## Status

This system is production ready but under active development.  At various fronts we are working to replace reliance on Puppet Agent and legacy MCollective, the project lives on [GitHub](https://github.com/choria-io).

Extensive performance testing has been done that showed the system to be stable at over 100 000 nodes.  Getting to 50 000 nodes is easily achievable using a single middleware compute node.
