+++
title = "Requirements"
weight = 105
toc = true
+++

For MCollective to function we need a few infrastructure components, this guide takes you through setting up all of these:

  * Set up a middleware broker using [NATS.io](https://nats.io/)
  * Configure server locations using DNS or manually if not using defaults
  * Configure MCollective
  * Create your first user

## Deployment Overview

Deploying MCollective with Choria is broken into the following main steps that the menu on the right guides you through:

  * Decide your desired NATS middleware topology and deploy it
  * If required configure DNS records or manual host locations
  * Configure MCollective using the [ripienaar/mcollective_choria](https://forge.puppet.com/ripienaar/mcollective_choria) module
  * Create your first users and their authorization rules
  * Explore the features Choria enable and read about the overall operation of MCollective on the official website with links this guide provide

There are a few optional additional features you can enable.

To make it to the end of this guide you will need to be able to effect root level changes to all your nodes via Puppet and Hiera.  You might have to change your your firewalls and DNS.

### Required

  * You must use Puppet 4 deployed using the Puppet Inc AIO packages - the one called _puppet-agent_
  * You must be using a Puppet Master based setup, typically using _puppetserver_
  * Your mcollective _server.cfg_ and _client.cfg_ should be Factory Default
  * You need to run middleware, Choria works best with NATS and provides a module to install that for you
  * Your certnames must match your FQDNs - the default
  * You need the [ripienaar/mcollective_choria](https://forge.puppet.com/ripienaar/mcollective_choria) module and all it's dependencies

### Optional

  * An optional PuppetDB integration exist to use PuppetDB as the source of truth.  This requires PuppetDB and extra configuration
  * Puppet Applications are supported and deployment can be done using Choria, it requires specific setup of your _puppetserver_
