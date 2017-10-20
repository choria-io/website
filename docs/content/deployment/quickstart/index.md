+++
title = "Quickstart"
weight = 107
toc = true
+++

{{% notice info %}}
The following is beneficial for sites with only 1 Puppet master and 1 PuppetDB. Consult further documentation if you have a more complex setup.
{{% /notice %}}

## Include Required Modules

```ruby
# Puppetfile
mod 'choria/mcollective_choria', '0.0.25'
mod 'choria/mcollective', '0.0.26'
mod 'choria/mcollective_agent_filemgr', '1.1.0'
mod 'choria/mcollective_agent_package', '4.5.0'
mod 'choria/mcollective_agent_puppet', '1.13.0'
mod 'choria/mcollective_agent_service', '3.1.4'
mod 'choria/mcollective_util_actionpolicy', '2.2.0'
mod 'choria/nats', '0.0.8'
mod 'camptocamp/systemd', '0.4.0'
```

## Associate the NATS Middleware

```puppet
node puppet.example.net {
  ...
  class{"nats": }
}
```
## Ensure All Nodes Get the New Configuration

```puppet
node "server1.example.net" {
  include mcollective
}
```

or if using _lookup()_

```yaml
# common.yaml
classes:
  - mcollective
  ...
```

## Update Hiera

```yaml
# common.yaml
...
mcollective_choria::config:
  use_srv_records: false
  puppetserver_host: "puppet.example.net"
  puppetserver_port: 8140
  puppetca_host: "puppet.example.net"
  puppetca_port: 8140
  puppetdb_host: "puppetdb.example.net"
  puppetdb_port: 8081
  middleware_hosts: "puppet.example.net:4222"
...
```

## Create a Client

Select a server you which to run commands from and create a user and include the client commands.

```yaml
# nodes/bastion.example.net.yaml
mcollective::client: true
```

Ensure puppet has successfully run, then create a user.

```bash
$ whoami
rip
$ mco choria request_cert
```
On Puppetserver:

```bash
$ puppet cert --sign rip.mcollective
```

## Authorize New User

Before the user can execute commands on remote hosts, we must grant permission.

```yaml
# common.yaml
...
mcollective::site_policies:
  - action: "allow"
    callers: "choria=rip.mcollective"
    actions: "*"
    facts: "*"
    classes: "*"
...
```
