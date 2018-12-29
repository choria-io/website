---
title: "Puppet 6 Support"
date: 2018-12-01T10:38:42+01:00
tags: ["releases", "server", "puppet"]
---

Back in July 2018 Puppet Inc officially announced that The Marionette Collective was being deprecated and will not be included in the future Puppet Agent releases.

This presented a problem for us as we relied on this packaging to install mcollective, services and its libraries.  We would now have to do all this ourselves.

At the same time I was working on the Choria Server and giving it backward compatibility capabilities (still in progress to hit 100%) so we couldn't support Puppet 6 on release day.

Today we published a bunch of releases and as of version 0.12.0 of the [choria/choria](http://forge.puppet.com/choria/choria) release we support Puppet 6 out of the box.

<!--more-->

There are some caveats to note:

 * On Puppet AIO 6 you will now automatically be switched to Choria Server, mcollectived does not exist anymore
 * On Puppet AIO 6 you will now automatically get a choria-release repository added, controllable using choria::manage_package_repo
 * If you are on a mix Puppet 5 and 6 environment you need to set JSON encoding [as per the deployment docs](https://choria.io/docs/deployment/mcollective/#puppet-6).
 * On Puppet AIO 6 you will see a new gem [choria-mcorpc-support](https://rubygems.org/gems/choria-mcorpc-support) installed into the AIO Ruby
 * Windows is not yet supported
 * Read the [pre-release documentation](https://choria.io/docs/configuration/choria_server/) for Choria Server keeping in mind limitations mentioned there

## Legacy mcollective source code

As you might know I sold the old Marionette Collective source code to Puppet Inc back in 2009, around 2014 already they announced it's being deprecated and for a long while I have attempted to get this code back from Puppet.

Recently this has happened and Puppet Inc have donated the whole project back to me including websites, mailing lists, plugins, domains and source code.

I've parked all of this on GitHub in a [choria-legacy](https://github.com/choria-legacy) project and archived it all, future work will happen on new forks.

Documentation from there have been incorporated and updated for Choria and the new [choria-mcorpc-support](https://rubygems.org/gems/choria-mcorpc-support) is built on the legacy mcollective source code - minus about 30 000 lines!