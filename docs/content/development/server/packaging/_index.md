+++
title = "Custom Packaging"
weight = 10
+++

The packaging used to build the open source Choria builds can also be used to build your own custom binaries.  You can customize many settings including names, paths, limits and security settings.  You can also plugin in various additional plugins that can augment or replace stock behaviors.

## Build Specification

Every Choria build needs a build specification, below we'll show one that build a custom Choria called acme-choria with custom paths and settings. This file goes into `packager/buildspec.yaml`.

```yaml
---
# Flags are variables that can be set by the linker to particular values. These are all String values, if you are linking
# in your own code you can add variables for those here too, below is the current set of Choria build time settables.  More
# about their individual uses later in this guide in sections dedicated to types of plugin
flags_map:
  TLS: github.com/choria-io/go-choria/build.TLS
  maxBrokerClients: github.com/choria-io/go-choria/build.maxBrokerClients
  Secure: github.com/choria-io/go-choria/vendor/github.com/choria-io/go-protocol/protocol.Secure
  Version: github.com/choria-io/go-choria/build.Version
  SHA: github.com/choria-io/go-choria/build.SHA
  BuildTime: github.com/choria-io/go-choria/build.BuildDate
  ProvisionBrokerURLs: github.com/choria-io/go-choria/build.ProvisionBrokerURLs
  ProvisionModeDefault: github.com/choria-io/go-choria/build.ProvisionModeDefault
  ProvisionAgent: github.com/choria-io/go-choria/build.ProvisionAgent
  ProvisionSecure: github.com/choria-io/go-choria/build.ProvisionSecure
  ProvisionRegistrationData: github.com/choria-io/go-choria/build.ProvisionRegistrationData
  ProvisionFacts: github.com/choria-io/go-choria/build.ProvisionFacts
  ProvisionToken: github.com/choria-io/go-choria/build.ProvisionToken

# Each build needs a name, this is the acme build
acme:
  # Each build will need binaries built, these are the description of binaries, flags, options
  # etc that should be built and for what OS and Architecture
  compile_targets:
    defaults:
      # The binary that will be built, this is temporary and will not be seen by end users
      output: choria-{{version}}-{{os}}-{{arch}}

      # Commands to run before build, you probably want to have at least these but can add your own
      pre:
        - rm additional_agent_*.go || true
        - rm plugin_*.go || true
        - go generate

      # Commands to run after a build, be careful if you run the built binary like here when doing
      # cross compiles you won't always be able to run the binary on the build host
      post:
        - {{output}} building

      # During compile you set the flags you want based on the flag map above, lets increase the broker
      # clients as an example.  Remember all of these have to be strings.
      flags:
        maxBrokerClients: "100000"

    # You can name your binary targets anything you like, see `go tool dist list` for a list of valid
    # OS and Arch, the system will cross compile to any of those for you.
    64bit_linux:
      os: linux
      arch: amd64

  # This is where we build packages for different operating systems.  Today the packaging system supports
  # building RPM and Dep packages
  packages:
    defaults:
      name: acme-choria
      bindir: /opt/acme/sbin
      etcdir: /opt/acme/etc/choria
      release: 1
      manage_conf: 1
      contact: ops@example.net
      rpm_group: System Environment/Base
      server_start_runlevels: "-"
      server_start_order: 50
      broker_start_runlevels: "-"
      broker_start_order: 50

    # Unique name for your package, this one will use the el/el7 template that you can see
    # under packager/templates/ in the go-choria repository.  It uses the 32bit_linux binary
    # we built earlier.
    el7_64:
      template: el/el7
      dist: el7
      target_arch: x86_64
      binary: 64bit_linux
      # This setting will override the contact set above, any of the settings can be set for
      # either all packages or one
      contact: rpm-packaging@example.net

    bionic_64:
      template: debian/generic
      target_arch: x86_64-linux-gnu
      binary: 64bit_linux
      contact: deb-packaging@example.net
```

## Additional Plugins

If you have any plugins to include in the resulting build you create `packaging/user_plugins.yaml`, in this example we link in a the [Choria Provisioning Agent](https://github.com/choria-io/provisioning-agent).

```yaml
---
choria_provision: github.com/choria-io/provisioning-agent/agent
```

You have to arrange for these to be available in the build path, see the example later in the doc.

## Building

You will need a node with Docker installed, it will download some containers to execute the builds and rpm packaging in etc. Basically you just need to run `rake build` or `rake build_binaries` if all you want are the binaries.

There are a few variables you can use to adjust things for what packages to build, as we've renamed everything here we will need to set a few things:

|Variable|Description|
|--------|-----------|
|VERSION |The Version to build, it would default to *development*, you can pick whatever you like|
|BUILD|Which build to execute, here we need to run the *acme* build|
|PACKAGES|Which packages to build, we'd need to say we want the *el7_64* and the *bionic_64*|

So to build full set of packages and binaries we do:

```bash
$ VERSION=1.2.3-acme BUILD=acme PACKAGES=el7_64,bionic_64 rake build
```

Or just the binaries

```bash
$ VERSION=1.2.3-acme BUILD=acme rake build
```

In my build where I fetch my own agents and add additional bits and rename the packages I use the Rakefile below to drive this process, it's a bit messy and inevitably yours will look differently.

```ruby
require "tmpdir"

GOPATH = File.expand_path("go")
CHORIA_DIR = File.join(GOPATH, "src/github.com/choria-io")
GOCHORIA_DIR = File.join(CHORIA_DIR, "go-choria")
VERSION = ENV.fetch("PKG_VERSION", ENV["CHORIA_TAG"])
TARGETS = ENV.fetch("TARGETS", "el7_64,bionic_64")
FLAVOURS = ENV.fetch("FLAVOURS", "acme")

ENV["PATH"] = "%s:/usr/local/go/bin:%s/bin" % [ENV["PATH"], GOPATH]
ENV["GOROOT"] = "/usr/local/go"
ENV["GOPATH"] = GOPATH

desc "Fetch choria source code"
task :fetch_choria do
  mkdir_p CHORIA_DIR
  Dir.chdir(CHORIA_DIR) do
    sh "git clone https://github.com/choria-io/go-choria.git"

    Dir.chdir(GOCHORIA_DIR) do
      sh "git checkout %s" % ENV["CHORIA_TAG"]
      sh "go get github.com/Masterminds/glide"
      sh "glide get --non-interactive --quick github.com/Masterminds/semver"
      sh "glide up"
      sh "glide install"
    end
  end
end

desc "Fetch extra agents to compile in"
task :fetch_agents do
  Dir.chdir(GOCHORIA_DIR) do
    mkdir_p "vendor/github.com/myorg"
    Dir.chdir("vendor/github.com/myorg") do
      sh "git clone https://github.com/myorg/my_plugin.git"
    end
  end
end

desc "Builds the acme-choria binary"
task :build do
  unless ENV["NO_FETCH"]
    rm_rf "go"

    Rake::Task[:fetch_choria].execute
    Rake::Task[:fetch_agents].execute
  end

  cp "user_plugins.yaml", File.join(GOCHORIA_DIR, "packager")
  cp "buildspec.yaml", File.join(GOCHORIA_DIR, "packager")

  Dir.chdir(GOCHORIA_DIR) do
    FLAVOURS.split(",").each do |flavour|
      sh "BUILD=%s VERSION=%s%s PACKAGES='%s' GOVERSION='1.10' rake build" % [flavour, VERSION, flavour, TARGETS]
    end
  end

  cp Dir.glob(File.join(GOCHORIA_DIR, "*.rpm")), "."

  sh "ls -l *rpm"
end
```

I can now build custom builds that copies in my *user_plugins.yaml* and *buildspec.yaml* into a fresh Choria checkout and builds it all:

```
CHORIA_TAG=0.8.0 PKG_VERSION=0.8.1-acme rake build
```

The result will be Choria 0.8.0 with my customisations called *acme-choria-0.8.1-acme.*.rpm*
