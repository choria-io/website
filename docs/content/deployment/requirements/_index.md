+++
title = "Requirements"
weight = 105
toc = true
+++

## Deployment Overview

Deploying Choria is broken into the following main steps that the menu on the left guides you through:

  * Decide on your desired Choria Broker middleware topology and deploy it
  * If required configure DNS records or manual host locations
  * Configure Choria using the [choria/choria](https://forge.puppet.com/choria/choria) and related modules
  * Create your first users and their authorization rules
  * Explore the features Choria enables and read about the overall operation on the official website with links this guide provides
  * Sign up for the [community](https://github.com/choria-io/general/discussions) to receive updates about the project

There are a few optional additional features you can enable.

To make it to the end of this guide you will need to be able to effect root level changes to all your nodes via Puppet and Hiera.  You might have to change your firewalls and DNS.

### Required

  * You must use Puppet 6 deployed using the Puppet Inc AIO packages - the one called _puppet-agent_.
  * You must be using a Puppet Server based setup, typically using _puppetserver_
  * You need to run middleware, Choria has its own Choria Broker that supports RedHat 7 and 8, Debian and Ubuntu.
  * Your certnames must match your FQDNs - the default
  * You need the [choria/choria](https://forge.puppet.com/choria/choria) module and all its dependencies

### Optional

  * An optional PuppetDB integration exist to use PuppetDB as the source of truth.  This requires PuppetDB and extra configuration
  * Playbooks can be written using the experimental Puppet Plans DSL.  This requires additional installation.
