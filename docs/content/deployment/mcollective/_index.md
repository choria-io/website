+++
title = "MCollective"
weight = 130
toc = true
+++

Configuring MCollective with Choria is generally very simple and involves just including 1 module and setting some Hiera data, it takes care of the entire process for you.

In MCollective terminology a _client_ is one you manage your network from - where you run _mco_ commands - and a _server_ is a node being managed.

## Every node

All nodes should have the _choria-mcollective_ module on them, by default every node becomes a MCollective Server ready to be managed via MCollective:

{{% notice tip %}}
The choria/mcollective_choria module has a number of [dependencies](https://forge.puppet.com/choria/mcollective_choria/dependencies), if you use R10k to manage environments please be sure to fetch all dependencies.
{{% /notice %}}


```puppet
node "server1.example.net" {
  include mcollective
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
mcollective::server: false
```

## Puppet 6

With the release of Puppet 6 Puppet Inc has [deprecated Marionette Collective](https://choria.io/mcollective), Choria support Puppet 6 by enabling the Choria Server as a replacement for `mcollectived`.

{{% notice warning %}}
Puppet 6 support is available since release `0.12.0` of the `choria/choria` module, it's the first release that supports Puppet 6 and while extensive testing was done please ensure you test this upgrade in your lab first.
{{% /notice %}}

If you are running a Puppet 6 only network everything will just work, if you run a mix 5 and 6 network you have to make create an additional data item:

```yaml
mcollective_choria::config:
  security.serializer: "json"
```

You can enable the same Choria Server on your Puppet 5 nodes by following the steps in the [Choria Server](/docs/configuration/choria_server/) guide.
