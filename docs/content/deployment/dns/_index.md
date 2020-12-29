+++
title = "DNS Setup"
weight = 120
toc = true
+++

By default as per Puppet behaviour the Puppet Server, Puppet CA and Choria Brokers are all found on the name _puppet_.  If you are doing a single node Broker installation on the Puppet Server called _puppet_ you do not need to configure anything and can continue to the next page.

When not using _puppet_ you can configure these settings manually but we strongly suggest you use SRV records if at all possible.

## Choria Brokers

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

## Custom Domain

By default these SRV records will be looked for in your machine's _domain_ fact, but you can customize this by creating data in your _Hiera_:

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

{{% notice tip %}}
At the moment there is some redundancy and confusion between mcollective and choria modules, we will merge this into one soon but kept it this way to disrupt users as little as possible
{{% /notice }}

```yaml
mcollective_choria::config:
  use_srv_records: false
  puppetserver_host: "puppet1.example.net"
  puppetserver_port: 8140
  puppetca_host: "ca1.example.net"
  puppetca_port: 8140
  puppetdb_host: "pdb1.example.net"
  puppetdb_port: 8081
  middleware_hosts: "choria1.example.net:4222,choria2.example.net:4222,choria3.example.net:4222"
```

You'll also need to configure the server to connect to your specific middleware:

```yaml
choria::server_config:
  plugin.choria.puppetserver_host: "puppet1.example.net"
  plugin.choria.puppetserver_port: 8140
  plugin.choria.puppetca_host: "ca1.example.net"
  plugin.choria.puppetca_port: 8140
  plugin.choria.puppetdb_host: "pdb1.example.net"
  plugin.choria.puppetdb_port: 8081
  plugin.choria.middleware_hosts: "choria1.example.net:4222,choria2.example.net:4222,choria3.example.net:4222"
```
