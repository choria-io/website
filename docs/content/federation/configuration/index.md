+++
title = "Configuration"
weight = 100
+++

## Middleware Sizing

A quick note about sizing, in the network diagrams we have 5 node Network Broker clusters in the Collectives and 3 in the Federation.  This is a suggestion only and what is right depends on your needs.

As mentioned a single Network Broker broker can easily handle 50 000 MCollective Daemons.  You almost never need a full 5 node cluster, 3 should be fine.  If you don't need it to be 100% always available even 1 node will do.

{{% notice tip %}}
It is best to run your Federation Broker Instances near the Collective they serve on your network to gain maximal benefits from the offloading the Client does onto the Federation Network Broker Cluster. Since they are integrated with the Network Broker I suggest simply enabling the feature on your existing brokers that serve each Collective.
{{% /notice %}}

Generally on the Federation side you do not need many nodes, the choice of 3 is mainly about reliability.  Network Brokers require an uneven numbered node count clusters.

![Federation Broker DNS](../../federation_dns_config.png)

## DNS

Like the normal MCollective Server and Client configuration is largely done via SRV records.  You can of course manually configure it but with SRV records it will more or less just work.

On the Collective side of the Broker the same configuration is used as for the MCollective Daemon and Client, so if you did the SRV setup for those you can just leave that alone.

On the Federation side you need additional records in the same domain:

```dns
_mcollective-server._tcp            IN      SRV     0       0       4222    choria1.ldn.example.net.
                                    IN      SRV     0       0       4222    choria2.ldn.example.net.
                                    IN      SRV     0       0       4222    choria3.ldn.example.net.
                                    IN      SRV     0       0       4222    choria4.ldn.example.net.
                                    IN      SRV     0       0       4222    choria5.ldn.example.net.

_mcollective-federation_server._tcp IN      SRV     0       0       4222    choria1.fed.example.net.
                                    IN      SRV     0       0       4222    choria2.fed.example.net.
                                    IN      SRV     0       0       4222    choria3.fed.example.net.
```

## Federation Broker Instance

Most likely you will run the Federation Brokers on the same machines as your Network Brokers clusters.  I suggest you name them something descriptive like here.

Here you see the Puppet code needed to start the Network Broker and Federation Broker Cluster called *london*. By default this will mean 50 instances of the Federation Broker and it will be capable of serving tremendously large Collectives.  You'll only see one process - `choria broker`.

```puppet
node "nats1.ldn.example.net" {
  class{"choria":
    srv_domain => "ldn.example.net"
  }

  class{"choria::broker":
    network_broker => true,
    federation_broker => true,
    federation_cluster => "london",
    network_peers => [
      "nats://choria1.ldn.example.net:5222",
      "nats://choria2.ldn.example.net:5222",
      "nats://choria3.ldn.example.net:5222",
      "nats://choria4.ldn.example.net:5222",
      "nats://choria5.ldn.example.net:5222"
    ],
  }
}
```

The Federation Broker statistics will be part of the normal Prometheus stats exposed on port `8222` and they will look up their SRV records in *ldn.example.net* as per the diagram.

## MCollective Client in the Federation

The only real additional configuration you should do is to tell it about all the default Federation Brokers with you can do in Hiera:

```yaml
mcollective::client_config:
  "plugin.choria.federation.collectives": "tokyo, london, new_york"
```

Once this setting is present it will become a Federated Client and use the *_mcollective-federation_server._tcp* SRV record or *plugin.choria.federation_middleware_hosts* configuration entry.

If these are absent it will fall back to the usual configuration and use the exact same middleware configuration via SRV or config as a normal client. This way you can optimise your config for simplicity or have a case where one client is usually connected to the local Collective and only sometimes Federated and in those cases different middleware is used.

If you have a client that is generally only connected to the local Collective or one where you do not always want to specify all the Federation Collectives you can use the *CHORIA_FED_COLLECTIVE* environment variable to set the *plugin.choria.federation.collectives* setting in the same format.

```bash
$ MCOLLECTIVE_FED_COLLECTIVE=tokyo mco puppet status
```

The above command will address only the *tokyo* Collective, or if it was a unfederated client it will become Federated with *tokyo* being the only known collective.  The same comma separated format is usable here.

You can use the *mco choria show_config* command to view your active configuration which will include sections about the Federation.

## Collective

No additional configuration is needed in your Member Collectives, they are just standalone as always.  As long as they use the same CA it will all work.
