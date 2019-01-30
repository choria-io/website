+++
title = "DNS Setup"
weight = 120
toc = true
+++

By default as per Puppet behaviour the Puppet Master, Puppet CA and NATS brokers are all found on the name _puppet_.  If you are doing a single node NATS installation on the Puppet Master called _puppet_ you do not need to configure anything and can continue to the next page.

When not using _puppet_ you can configure these settings manually but we strongly suggest you use SRV records if at all possible.

## NATS Brokers

You can configure where your NATS brokers live using these SRV records:

```dns
_x-puppet-mcollective._tcp   IN  SRV 10  0 4222  nats1.example.net.
_x-puppet-mcollective._tcp   IN  SRV 11  0 4222  nats2.example.net.
_x-puppet-mcollective._tcp   IN  SRV 12  0 4222  nats3.example.net.
```

This means you have 3 of them and they all listen on port _4222_.

## Puppet and Puppet CA

If your Puppet CA, PuppetDB and Puppet Server are all on the same host, you can configure that all with a single SRV record that is compatible with Puppet SRV setup.

```dns
_x-puppet._tcp               IN  SRV 10  0 8140  puppet1.example.net.
```

But if you wish to split the CA and DB from the master add these:

```dns
_x-puppet-ca._tcp            IN  SRV 10  0 8140  puppetca1.example.net.
_x-puppet-db._tcp            IN  SRV 10  0 8081  puppetdb1.example.net.
```

## Discovery Proxy

If you use the Discovery Proxy you can configure this in SRV records:

```dns
_mcollective-discovery._tcp            IN  SRV 10  0 8085  puppetdb1.example.net.
```

## Custom Domain

By default these SRV records will be looked for in your machines _domain_ fact, but you can customize this by creating data in your _Hiera_:

```yaml
mcollective_choria::config:
  srv_domain: "prod.example.net"
```

## Disabling SRV support

You might be in a situation where you have multiple environments like development and production in the same domain.  You might want to use SRV for production but not for development.

```yaml
mcollective_choria::config:
  use_srv_records: false
```

## Manual Config

If you have to you can configure these locations manually by creating _Hiera_ data:

```yaml
mcollective_choria::config:
  use_srv_records: false
  puppetserver_host: "puppet1.example.net"
  puppetserver_port: 8140
  puppetca_host: "ca1.example.net"
  puppetca_port: 8140
  puppetdb_host: "pdb1.example.net"
  puppetdb_port: 8081
  discovery_host: "pdb1.example.net"
  discovery_port: 8085
  middleware_hosts: "nats1.example.net:4222,nats2.example.net:4222,nats3.example.net:4222"
```

If you're using the choria server, you'll also need to configure the server to connect to your specific middleware:

```yaml
choria::server_config:
  classesfile: "/opt/puppetlabs/puppet/cache/state/classes.txt"
  rpcaudit: 1
  plugin.rpcaudit.logfile: "/var/log/choria-audit.log"
  plugin.yaml: "/etc/puppetlabs/mcollective/generated-facts.yaml"
  plugin.choria.agent_provider.mcorpc.agent_shim: "/usr/bin/choria_mcollective_agent_compat.rb"
  plugin.choria.agent_provider.mcorpc.config: "/etc/puppetlabs/mcollective/choria-shim.cfg"
  plugin.choria.agent_provider.mcorpc.libdir: "/opt/puppetlabs/mcollective/plugins"
  plugin.choria.puppetserver_host: "puppet1.example.net"
  plugin.choria.puppetserver_port: 8140
  plugin.choria.puppetca_host: "ca1.example.net"
  plugin.choria.puppetca_port: 8140
  plugin.choria.puppetdb_host: "pdb1.example.net"
  plugin.choria.puppetdb_port: 8081
  plugin.choria.discovery_host: "pdb1.example.net"
  plugin.choria.discovery_port: 8085
  plugin.choria.middleware_hosts: "nats1.example.net:4222,nats2.example.net:4222,nats3.example.net:4222"
```
