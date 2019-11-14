+++
title = "Gem Distribution"
toc = true
weight = 240
+++

Choria requires certain ruby gems to be installed for Puppet's ruby and will install these via the internet by default. However, many sites have policies prohibiting their nodes from accessing dependencies via the internet. Choria allows you to manage its own dependencies and disable the built in Gem management:

First, configure Choria to not install any gem dependencies via _Hiera_:

```yaml
mcollective_choria::manage_gem_dependencies: false
```

You can then configure Choria to install the gems for you via the system packager (see below for how to package gems up). Let's say you called the package _puppet-gem-choria-mcorpc-support_, you can configure installation again via _Hiera_:

```yaml
mcollective_choria::package_dependencies:
  "puppet-gem-choria-mcorpc-support": "2.20.7"
```

It will now install this package for you and allow you to manage the version of it. The ordering is handled correctly and MCollective will restart appropriately.


## How to package gems with fpm

You can package gems into a package for your system package manager using using [fpm](https://fpm.readthedocs.io/en/latest/). To ensure you package not only the gem itself but all of its dependencies, follow the [section on packaging gem dependencies](https://fpm.readthedocs.io/en/latest/source/gem.html#packaging-individual-dependencies). 

### Example: packaging choria-mcorpc-support into an rpm

This example shows how to package the `choria-mcorpc-support` gem and its dependencies into RPM packages. `fpm` will automatically make the `choria-mcorpc-support` RPM depend on the other RPM packages.
Note that this example requires an enterprise linux host with the necessary RPM tools installed.

1. Prepare a staging directory
   ```
   mkdir /tmp/gems
   ```
1. Download the gem and its dependencies
   ```
   gem install --no-ri --no-rdoc --install-dir /tmp/gems choria-mcorpc-support
   ```
1. Package each gem into an RPM called "puppet-gem...". Note that this assumes that Puppet's `gem` binary is at `/opt/puppetlabs/puppet/bin/gem`
   ```
   find /tmp/gems/cache -name '*.gem' | xargs -rn1 -I {} fpm \
   --description "Packaged puppet Gem: {}" \
   --gem-gem "/opt/puppetlabs/puppet/bin/gem" \
   --no-gem-env-shebang \
   --gem-package-name-prefix "puppet-gem" \
   -s gem \
   -t rpm {}
   ```
1. Copy the RPMs to your RPM repository (in this example: `/mnt/repo/choria`)
   ```
   mv *.rpm /mnt/repo/choria`
   ```
1. Refresh the repo metadata
   ```
   createrepo /mnt/repo/choria`
   ```
