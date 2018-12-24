+++
title = "Configuration"
weight = 200
+++

## Middleware Sizing

A single Choria Network Broker can easily handle 50 000 Choria managed compute nodes. You almost never need more than 3 brokers. If you don't need it to be 100% always available we recommend a single Network Broker per Collective.

Generally on the Federation side you do not need many brokers, the choice of 3 is mainly about reliability and again if you do not need 100% reliability, a single node is recommended.  Network Brokers require an uneven numbered node count clusters.  Internally the Federation Brokers use a worker pool of 10 Brokers consisting of 3 micro services each.  This is very scalable and should be able to handle any reasonable size network.

![Federation Broker DNS](../../federation_dns_config.png)

{{% notice tip %}}
It is best to run your Federation Broker Instances near the Collective they serve on your network to gain maximal benefits from the offloading the Client does onto the Federation Network Broker Cluster. Since they are integrated with the Network Broker I suggest simply enabling the feature on your existing brokers that serve each Collective.
{{% /notice %}}

## DNS SRV Records

Like the normal Choria Server and Client configuration is largely done via SRV records.  You can of course manually configure it but with SRV records it will more or less just work.

Below you'll see DNS records for the 2 SRV Records covering the Federation (left, numbered **1**) and the Collective (right, numbered **2**). These are in the domain configured as `srv_domain` to the `choria` class - `ldn.example.net` in the example below.  We are configuring the middle 3 Federation Brokers.

To discover the middleware for the Federation:

```nohighlight
_mcollective-federation_server._tcp IN  SRV 0 0 4222  choria1.fed.example.net.
                                    IN  SRV 0 0 4222  choria2.fed.example.net.
                                    IN  SRV 0 0 4222  choria3.fed.example.net.
```

To discover the middleware for the Collective:

```nohighlight
_mcollective-server._tcp            IN  SRV 0 0 4222  choria1.ldn.example.net.
                                    IN  SRV 0 0 4222  choria2.ldn.example.net.
                                    IN  SRV 0 0 4222  choria3.ldn.example.net.
```

{{% notice tip %}}
If you cannot or do not wish to use use SRV records see later in the same page for manual configuration
{{% /notice %}}

## Federation Broker Instance

Most likely you will run the Federation Brokers on the same machines as your Network Brokers. Here you see the Puppet code needed to start the Network Broker and Federation Broker Cluster called *london*. By default this will mean 30 instances of the Federation Broker - 10 per Broker -and it will be capable of serving tremendously large Collectives.  You'll only see one process - `choria broker`.

{{% notice tip %}}
While I show the Puppet code here for completeness, I recommend using Hiera to configure these settings
{{% /notice %}}

```puppet
node "nats1.ldn.example.net" {
  class { "choria":
    srv_domain => "ldn.example.net"
  }

  class { "choria::broker":
    network_broker     => true,
    federation_broker  => true,
    federation_cluster => "london",

    network_peers => [
      "nats://choria1.ldn.example.net:4223",
      "nats://choria2.ldn.example.net:4223",
      "nats://choria3.ldn.example.net:4223",
    ]
  }
}
```

The Federation Broker statistics will be part of the normal Prometheus stats exposed on port `8222` and they will look up their SRV records in *ldn.example.net* as per the diagram.

If you wish to configure the middleware manually rather than SRV records you can do so:

```puppet
  class{"choria::broker":
    network_broker     => true,
    federation_broker  => true,
    federation_cluster => "london",

    federation_middleware_hosts => [
      "choria1.fed.example.net:4222",
      "choria2.fed.example.net:4222",
      "choria3.fed.example.net:4222",
    ],
    collective_middleware_hosts => [
      "choria1.ldn.example.net:4222",
      "choria2.ldn.example.net:4222",
      "choria3.ldn.example.net:4222",
    ],
    network_peers => [
      "nats://choria1.ldn.example.net:4223",
      "nats://choria2.ldn.example.net:4223",
      "nats://choria3.ldn.example.net:4223",
    ]
  }
```

The full reference of Federation related configuration options can be seen below:

|Option|Description|Sample|
|------|-----------|------|
|plugin.choria.federation.collectives|List of collectives that belong to the federation|`london,tokyo,new_york`|
|plugin.choria.federation_middleware_hosts|Choria Brokers on the Federation side to connect to|`c1.fed.example.net:4222,c2.fed.example.net:4222`|
|plugin.choria.federation.cluster|A cluster name that a specific Federation Broker forms part of|`london`|

## Choria Client in the Federation

The only real additional configuration you should do is to tell it about all the default Federation Brokers with you can do in Hiera:

```yaml
mcollective::client_config:
  "plugin.choria.federation.collectives": "tokyo, london, new_york"
```

Once this setting is present it will become a Federated Client and use the *_mcollective-federation_server._tcp* SRV record or *plugin.choria.federation_middleware_hosts* configuration entry.

If these are absent it will fall back to the usual configuration and use the exact same middleware configuration via SRV or config as a normal client. This way you can optimize your config for simplicity or have a case where one client is usually connected to the local Collective and only sometimes Federated and in those cases different middleware is used.

If you have a client that is generally only connected to the local Collective or one where you do not always want to specify all the Federation Collectives you can use the *CHORIA_FED_COLLECTIVE* environment variable to set the *plugin.choria.federation.collectives* setting in the same format.

```bash
$ MCOLLECTIVE_FED_COLLECTIVE=tokyo mco puppet status
```

The above command will address only the *tokyo* Collective, or if it was a unfederated client it will become Federated with *tokyo* being the only known collective.  The same comma separated format is usable here.

You can use the *mco choria show_config* command to view your active configuration which will include sections about the Federation.

## Collective

No additional configuration is needed in your Member Collectives, they are just standalone as always.  As long as they use the same CA it will all work.
