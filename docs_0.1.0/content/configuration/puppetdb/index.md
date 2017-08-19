+++
title = "PuppetDB Discovery"
toc = true
weight = 210
+++

Choria includes a [PuppetDB](https://docs.puppet.com/puppetdb/) based discovery plugin but it's not enabled by default.

This is an advanced PuppetDB plugin that is subcollective aware and supports node, facts with `dot notation`, class and agent filters. It uses the new _Puppet PQL_ under the hood and so requires a very recent PuppetDB.

Using it you get a very fast discovery workflow but without the awareness of which nodes are actually up and responding, it's suitable for situations where you have a stable network, or really care to know when known machines are not responding as is common during software deployments. It makes a very comfortable to use default discovery plugin.

## Requirements

There are 2 modes of deployment for you to choose from depending on your security needs:

### Clients communicate directly with PuppetDB

Your MCollective _client_ machine needs to be able to communicate with PuppetDB on its SSL port. The client will use the same certificates that was created using *mco choria request_cert* so you don't need to do anything with the normal Puppet client-tools config, though you might find setting those up helpful.

{{% notice warning %}}
Giving people access to PuppetDB in this manner will allow them to do all kinds of thing with your data as there are no ACL features in PuppetDB, consider carefully who you allow to connect to PuppetDB on any port.
{{% /notice %}}

### Clients communicate with the Discovery Proxy

A newly released feature provides a Proxy server between your client and PuppetDB.  This way the security concerns with the above approach is removed but at the expense of a more complex deployment.

Additionally you get the ability to make stored queries that you can then use like `mco service restart httpd -I set:acme_service`, where `acme_service` is a named Node Set running a stored PQL query.

The Discovery Proxy is a new project so I suspect there will be some bumps in the road, feel free to open issues on GitHub as you find them.

## Using

In general you can just go about using MCollective as normal after configuring it (see below).  All your usual filters like _-I_, _-C_, _-W_ etc all work as normal.

Your discovery should take a fraction of a second rather than the usual 2 seconds or more and will reflect what PuppetDB thinks it should be out there.

{{% notice tip %}}
Fact dot notation is supported from version 0.0.29 and newer
{{% /notice %}}

An additional benefit of this plugin is that you can use the new dot notation for using sctructured facts directly in discovery filters:

```bash
$ mco find -W "os.distro.id!=CentOS"
```

The dot notation is parsed by PuppetDB itself so you restricted to it's behaviours.


### PQL
There is an advanced feature that lets you construct complex queries using the [PQL language](https://docs.puppet.com/puppetdb/latest/api/query/v4/pql.html) for discovery though.

```bash
$ mco find -I "pql:nodes[certname] { certname ~ '^dev' }"
dev3.example.net
dev1.example.net
dev2.example.net
```

You can construct very complex queries that can match even to the level of specific properties of resources and classes:

```bash
$ mco find -I "pql:inventory[certname] { resources { type = 'User' and title = 'rip' and parameters.ensure = 'present'}}"
```

PQL queries comes in all forms, there are many examples at the Puppet docs. Though you should note that you must ensure you only ever return the certname as in the above example.

If you configure the Puppet Client Tools (see below) you can test these queries on the CLI:

```bash
% puppet query "inventory[certname] { resources { type = 'User' and title = 'rip' and parameters.ensure = 'present'}}"
[
  {
    "certname": "host1.example.net"
  }
]
```

Your queries **MUST** return the data as above - just the certname property.

### Node Sets

If you use the Discovery Proxy you can use node sets.

First configure the Discovery Proxy and MCollective as below, then create a Node Set:

```bash
$ discovery_proxy sets create mt_hosts
Please enter a PQL query, you can scroll back for history and use normal shell editing short cuts:
pql> inventory { facts.country = "mt" }
Matched Nodes:

   dev1.example.net           dev10.example.net          dev11.example.net
   dev12.example.net          dev13.example.net          dev2.example.net
   dev3.example.net           dev4.example.net           dev5.example.net
   dev6.example.net           dev7.example.net           dev8.example.net
   dev9.example.net           nuc1.example.net           nuc2.example.net

Do you want to store this query (y/n)> y
```

This creates a set *mt_hosts*, you can now use it in MCollective:

```bash
$ mco service restart httpd -I set:mt_hosts
```

## Configuring MCollective

You do not have to configure this to be the default discovery method, instead you can use it just when you need or want:

```bash
$ mco puppet status --dm=choria
```

By passing _--dm=choria_ to MCollective commands you enable this discovery method just for the duration of that command.  This is a good way to test the feature before enabling it by default.

You can set this discovery method to be your default by adding the following hiera data:

```yaml
mcollective::client_config:
  default_discovery_method: "choria"
```

By default it will attempt to find PuppetDB on puppet:8081, you can configure this [using DNS or manually](../../deployment/dns/).

## Configuring the Discovery Proxy

### Discovery Proxy Service

Make sure you have the *choria/discovery_proxy* Puppet Module, running the service on default port `8085` is simple:

```puppet
class{"discovery_proxy": }
```

This will install */usr/bin/discovery_service* that is both the proxy and the CLI for creating sets.

### Configuring MCollective for the Proxy

If you wish to use a Discovery Proxy you can rely on SRV DNS record for the host and port or configure manually.  In either event you have to enable the Proxy:

```yaml
mcollective::client_config:
  "plugin.choria.discovery_proxy": "true"
```

Then create the *_mcollective-discovery._tcp* SRV record:

```dns
_mcollective-discovery._tcp            IN  SRV 10  0 8085  puppetdb1.example.net.
```

Or set *plugin.choria.discovery_host* and *plugin.choria.discovery_port*:

```yaml
mcollective::client_config:
  "plugin.choria.discovery_proxy": "true"
  "plugin.choria.discovery_host": "puppetdb1.example.net"
  "plugin.choria.discovery_port": "8085"
```

{{% notice tip %}}
Use `mco choria show_config` to verify these settings are correct and what is found in DNS for server names
{{% /notice %}}

You can install the Discovery Proxy client tool using Puppet:

```puppet
class{"discovery_proxy":
  manage_service => false
}
```

You can now use the `discovery_proxy sets --help` command to maintain node sets.  This command uses the configuration you just created.

## Configuring Puppet (optional)

It's convenient to be able to query PuppetDB using the _puppet query_ command especially if you want to use the custom PQL based discovery, create ~/.puppetlabs/client-tools/puppetdb.conf with:

```json
{
  "puppetdb": {
    "server_urls": "https://puppet:8081",
    "cacert": "/home/rip/.puppetlabs/etc/puppet/ssl/certs/ca.pem",
    "cert": "/home/rip/.puppetlabs/etc/puppet/ssl/certs/rip.mcollective.pem",
    "key": "/home/rip/.puppetlabs/etc/puppet/ssl/private_keys/rip.mcollective.pem"
  }
}
```

And then install the puppet-client-tools package from the Puppet Labs repos.
