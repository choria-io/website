---
title: "Choria Server 0.9.0"
date: 2018-12-28T23:05:00+01:00
tags: ["releases", "server"]
---

Today I released version 0.9.0 of the Choria Server along with an update to the Ruby plugin for MCollective.

This is a significant milestone release that give us full support for custom Certificate Authorities including chains of Intermediates.  The [Choria Server Provisioner](https://github.com/choria-io/provisioning-agent) supports requesting CSR's from nodes and supplying those nodes with signed certs and you can integrate it with any CA with an API of your choosing.

We've also fixed some bugs, tweaked some things and generally iterated ever forward.

<!--more-->

## Certificate Authorities

So far we have relied on the Puppet CA for all certificate authority needs.  The Choria Server had some features to let you configure custom paths to SSL files but this was not really possible with the Ruby client.

In this release the Ruby client gained the ability to be configured with custom paths and both the server and client now support non Puppet CAs including chained ones.

Here's an example of the config you can supply for custom paths:

```ini
plugin.security.provider = file
plugin.security.file.certificate = /etc/choria/ssl/cert.pem
plugin.security.file.key = /etc/choria/ssl/private.pem
plugin.security.file.ca = /etc/choria/ssl/chain.pem
plugin.security.file.cache = /etc/choria/ssl/cache
```

Setting up your own CA chain and integrating that into Choria was documented on the [documentation site](https://choria.io/docs/configuration/custom_ca/).

The [Provisioning Agent](https://github.com/choria-io/provisioning-agent) have also had some doc updates showing how to integrate it into a CFSSL CA.

A lof ot this work was completed with the help of [Vincent Janelle](https://twitter.com/randomfrequency).

## Server plugin documentation

We started documenting how to write various kinds of plugins for the Choria Server and also how to compile your own custom packages using our tooling.  This documentation is in the [Extending Choria section](https://choria.io/docs/development/) of the documentation.

## Additional lifecycle events

Choria Server publishes life cycle events on startup, shutdown and once provisioning is complete.  In this release it also started publishing hourly messages to indicate it is still alive.  These messages can be used by other tools to detect events related to Choria - for example the Server Provisioner listens to startup events of unconfigured servers so they can be instantly configured.

The `alive` events will be used to create a service that keeps a tally of all known nodes and their versions so this information can be exposed to Prometheus.

Choria Lifecycle Events are created using the [go-lifecycle](https://github.com/choria-io/go-lifecycle) project, this project reached version 1.0.0 this cycle so we will support it subject to SemVer.

You can observe these events on your networks using the `choria tool event`.  If you want to build on these the schemas are published in our [schemas repository](https://github.com/choria-io/schemas/tree/master/choria/lifecycle/v1).  In Go you can use the SDK in the lifecycle project.

## Runtime Mutation of configuration defaults

We added a new plugin type that can be compiled into a Choria Server that let you adjust the default configuration of the server at runtime.

This is quite esoteric, I envision it can be used in cases like below:

 * You want to enable/disable provisioning mode on criteria entirely unique to your setup
 * You want to dynamically enable/disable PKI or TLS security. For instance some DCs might not have access to a CA
 * You want to look up hosts to connect to using means other than SRV or static configuration
 * You want to prepare a token or other kind of credential, perhaps fetched from credential store during initialization

An example doing the dynamic PKI/TLS thing is in the above mentioned plugin documentation

## Backward compatibility improved

Previously for users using the Choria Server `mco facts` was not working due to problems caused by the move to JSON serialization.  In this release we have fixed this and `mco facts` once again works.

This is exciting and all but really under the covers this means a whole class of problem is solved now, people who wrote their own agents that returned anything but basic data will benefit from increased backward compatibility.

