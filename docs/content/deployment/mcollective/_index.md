+++
title = "Choria"
weight = 130
toc = true
+++

Configuring Choria is generally very simple and involves just including 1 module and setting some Hiera data, it takes care of the entire process for you.

In Choria terminology a _client_ is one you manage your network from - where you run _mco_ commands - and a _server_ is a node being managed.

## Every node

All nodes should have the _choria_mcollective_ module on them, by default every node becomes an Choria Server ready to be managed via Choria:

{{% notice tip %}}
The choria/choria module has a number of [dependencies](https://forge.puppet.com/choria/choria/dependencies), if you use R10k to manage environments please be sure to fetch all dependencies.
{{% /notice %}}


```puppet
node "server1.example.net" {
  include choria
}
```

## Client nodes

On machines where you wish to run _mco_ commands like your Bastion nodes you have to configure them to be clients, you do this via _Hiera_:

```yaml
mcollective::client: true
```

If you wish to have Client Only machines, you can disable the server on them:

```yaml
mcollective::client: true
choria::server: false
```
