+++
title = "About Choria"
weight = 1
icon = "<b>1. </b>"
+++

Choria is a distribution of plugins and utilities that enhances [The Marionette Collective](https://docs.puppet.com/mcollective/) from Puppet Inc with modern abilities and significantly improves the getting started experience.

## Overview

MCollective is notoriously hard to install, the official installation guide takes you through pages and pages of information and expect you to make very important decisions that affects the security of your platform before you even know the system.

Choria exists to provide a distribution of plugins designed to be secure by default, easy to install and with no prior knowledge required to get to a point of having a secure production ready deployment of MCollective. Even a clustered setup of MCollective using Choria should take less than 1 hour.

After installing MCollective using Choria you can be sure that the security decisions taken are robust and the full Authentication, Authorization and Auditing feature set that sets MCollective apart from others have been configured for you.  Choria was written by the original architect of The Marionette Collective and represents current thinking on deployment and security of the system.

## Features

  * A complete playbook system that integrates MCollective with any number of other services to conduct full workflow based actions. YAML and Puppet based DSLs exist.
  * A Security Plugin utilizing the Puppet Certificate Authority system
  * A Discovery plugin for PuppetDB giving you a responsive and predictable interaction mode with support for PQL based infrastructure discovery
  * A Connector using NATS.io middleware that has been tested to 100s of thousands of nodes. 50 000 nodes can comfortably be run on a single NATS server.
  * Federations of Collectives eases the admin burden of managing large geographically distributed Collectives
  * Full end to end Authentication, Authorization and Auditing out of the box
  * Common Puppet eco system plugins like Package, Service, Puppet and File Manager deployed and ready to use
  * Operating System support for every OS Puppet supports.
  * Support for SRV records and sane configuration defaults to attain a zero config setup

See the [Deployment Guide](../deployment) for details on installation.

## Status

All the features of this module work as intended, though bugs are inevitable.  All Puppet supported Operating Systems are supported, including Windows, and the project is under active development on [GitHub](https://github.com/choria-io).

Extensive performance testing has been done that showed the system to be stable at over 100 000 nodes.  Getting to 50 000 nodes is easily achievable using a single middleware server.
