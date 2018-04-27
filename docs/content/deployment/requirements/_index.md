+++
title = "Requirements"
weight = 105
toc = true
+++

## Deployment Overview

Deploying MCollective with Choria is broken into the following main steps that the menu on the left guides you through:

  * Decide your desired Choria Broker middleware topology and deploy it
  * If required configure DNS records or manual host locations
  * Configure MCollective using the [choria/choria](https://forge.puppet.com/choria/choria) module
  * Create your first users and their authorization rules
  * Explore the features Choria enable and read about the overall operation of MCollective on the official website with links this guide provide
  * Sign up for the [community](https://groups.google.com/forum/#!forum/choria-users) to receive updates about the project

There are a few optional additional features you can enable.

To make it to the end of this guide you will need to be able to effect root level changes to all your nodes via Puppet and Hiera.  You might have to change your your firewalls and DNS.

### Required

  * You must use Puppet 5 deployed using the Puppet Inc AIO packages - the one called _puppet-agent_
  * If you want to use Playbooks you must use Puppet 5.4.0 at least
  * You must be using a Puppet Master based setup, typically using _puppetserver_
  * You need to run middleware, Choria has it's own Choria Broker that supports RedHat 5, 6, 7, Debian Stretch and Ubuntu Xenial
  * Your certnames must match your FQDNs - the default
  * You need the [choria/choria](https://forge.puppet.com/choria/choria) module and all it's dependencies

### Optional

  * An optional PuppetDB integration exist to use PuppetDB as the source of truth.  This requires PuppetDB and extra configuration
  * Playbooks can be written using the experimental Puppet Plans DSL.  This requires additional installation.
