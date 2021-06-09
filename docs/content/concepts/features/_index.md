+++
title = "Features"
weight = 201
toc = true
+++

## End User Features

  * A complete [Playbook System](/docs/playbooks/) that integrates Choria with any number of other services to conduct full workflow based multi step orchestrations.
  * Support for executing [Puppet Tasks](/docs/tasks/) as found on the Puppet Forge without the need for SSH and with RBAC for the Open Source Puppet user.
  * A Discovery plugin for PuppetDB giving you a responsive and predictable interaction mode with support for PQL based infrastructure discovery
  * Common plugins like Package, Service, Puppet and File Manager deployed and ready to use, many more actively maintained and ready to install.
  * Ability to construct [Autonomous Agents](../../autoagents) that can manage any device or component continuously.
  * Explore our current R&D project [Choria Scout](../../scout) that will deliver a full featured monitoring solution.

## Developer Features

  * Write your own Agents that expose new Actions unique to your environment in any language and interact with them from Playbooks, CLI and your own custom Clients.
  * [Embed a Choria Server into your own Golang software](https://github.com/ripienaar/embedded-choria-sample#readme) to create a fast and secure management backplane for any software
  * [Embed a Choria Server into your IoT platform](https://github.com/ripienaar/choriapi) to create a secure data access layer and life cycle management
  * Create highly scalable data pipe lines with node data being processed through Stream Processors like NATS Streaming using Choria Data Adapters, ideal for IoT or managing very large geo diverse server estates
  * Build vastly scalable asset data stores using Stream Processing techniques and Choria Metadata
  * Extend and enhance core capabilities with new Agent Providers and more using Golang and Ruby

## Operations Features

  * Integration with the Puppet Certificate Authority system or Kubernetes Certmanager.
  * A middleware technology that is used in production serving 100s of thousands of Choria Server. 50 000 server can comfortably be run on a single Choria Broker.
  * Federations of Collectives eases the admin burden of managing large geographically distributed Collectives
  * Full end to end Authentication, Authorization and Auditing out of the box
  * Support for SRV records and sane configuration defaults to attain a zero config setup
  * Integration with modern monitoring systems via extensive metrics being exposed using the Prometheus standard
  * Produce a custom build of Choria with auto provisioning, custom security and custom automation agents compiled right into the binary, using your own branding and paths without a reliance on Puppet


See the [Deployment Guide](/docs/deployment/) for details on installation.
