+++
title = "Requirements"
weight = 105
toc = true
+++

## Deployment Overview

Deploying MCollective with Choria is broken into the following main steps that the menu on the right guides you through:

  * Decide your desired [NATS](https://nats.io) middleware topology and deploy it
  * If required configure DNS records or manual host locations
  * Configure MCollective using the [choria/mcollective_choria](https://forge.puppet.com/choria/mcollective_choria) module
  * Create your first users and their authorization rules
  * Explore the features Choria enable and read about the overall operation of MCollective on the official website with links this guide provide
  * Sign up for the [community](https://groups.google.com/forum/#!forum/choria-users) to receive updates about the project

There are a few optional additional features you can enable.

To make it to the end of this guide you will need to be able to effect root level changes to all your nodes via Puppet and Hiera.  You might have to change your your firewalls and DNS.

### Required

  * You must use Puppet 4 or 5 deployed using the Puppet Inc AIO packages - the one called _puppet-agent_
  * You must be using a Puppet Master based setup, typically using _puppetserver_
  * You need to run middleware, Choria works best with NATS and provides a module to install that for you
  * Your certnames must match your FQDNs - the default
  * You need the [choria/mcollective_choria](https://forge.puppet.com/choria/mcollective_choria) module and all it's dependencies

### Optional

  * An optional PuppetDB integration exist to use PuppetDB as the source of truth.  This requires PuppetDB and extra configuration
  * Puppet Applications are supported and deployment can be done using Choria, it requires specific setup of your _puppetserver_
