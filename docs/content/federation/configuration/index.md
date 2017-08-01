+++
title = "Configuration"
weight = 100
+++

## Middleware Sizing

A quick note about sizing, in the network diagrams we have 5 node NATS clusters in the Collectives and 3 in the Federation.  This is a suggestion only and what is right depends on your needs.

As mentioned a single NATS broker can easily handle 2 000 MCollective Daemons and probably a lot more.  You almost never need a full 5 node cluster, 3 should be fine.  If you don't need it to be 100% always available even 1 node will do.

{{% notice tip %}}
It is best to run your Federation Broker Instances near the Collective they serve on your network to gain maximal benefits from the offloading the Client does onto the Federation Broker Cluster.
{{% /notice %}}

Generally on the Federation side you do not need many nodes, the choice of 3 is mainly about reliability.  NATS work best in uneven numbered node count clusters.

You can run as many Federation Broker Instances as you need, they are entirely stateless and automatically form clusters load sharing the work.

I suggest 1 or 2 instances for every node you run your NATS daemons on but even 1 will do fine. If you have many people using MCollective at the same time or people and automations on the same Collective then run at least a 2 or 3 per Collective.

The Federtion and Collective can even share NATS infrastructure, I imagine this is only useful in Development or Testing though.

![Federation Broker DNS](../../federation_dns_config.png)

## DNS

Like the normal MCollective Server and Client configuration is largely done via SRV records.  You can of course manually configure it but with SRV records it will more or less just work.

On the Collective side of the Broker the same configuration is used as for the MCollective Daemon and Client, so if you did the SRV setup for those you can just leave that alone.

On the Federation side you need additional records in the same domain:

```dns
_mcollective-server._tcp            IN      SRV     0       0       4222    nats1.ldn.example.net.
                                    IN      SRV     0       0       4222    nats2.ldn.example.net.
                                    IN      SRV     0       0       4222    nats3.ldn.example.net.
                                    IN      SRV     0       0       4222    nats4.ldn.example.net.
                                    IN      SRV     0       0       4222    nats5.ldn.example.net.

_mcollective-federation_server._tcp IN      SRV     0       0       4222    nats1.fed.example.net.
                                    IN      SRV     0       0       4222    nats2.fed.example.net.
                                    IN      SRV     0       0       4222    nats3.fed.example.net.
```

## Federation Broker Instance

Most likely you will run the Federation Brokers on the same machines as your NATS clusters.  I suggest you name them something descriptive like here:

{{% notice warning %}}
At present the *mcollective_choria::federation_broker* only supports *systemd* based Operating Systems
{{% /notice %}}

```puppet
node "nats1.ldn.example.net" {
  class{"nats":
    routes_password => "Vrph54FBcIvdM"
    servers => [
      "nats1.ldn.example.net",
      "nats2.ldn.example.net",
      "nats3.ldn.example.net",
      "nats4.ldn.example.net",
      "nats5.ldn.example.net"
    ],
  }

  mcollective_choria::federation_broker{"london":
    instances => 2,
    stats_base_port => 8000,
    srv_domain => "ldn.example.net"
  }
}
```

Here you see the Puppet code needed to start the Federation Broker Cluster called *london* with 2 Instances on every one of your NATS nodes.  Thus you'll have 5 x NATS nodes and 10 x Federation Broker Instances.

The Federation Broker Instance will listen on ports 8000 - 8005 for HTTP requests to */stats* and they will look up their SRV records in *ldn.example.net* as per the diagram.

### Using supervisord instead of systemd

By default the Federation Broker services will be started with *systemd*, managed by the *camptocamp/systemd*. If you don't have systemd, you can use *ajcrowe/supervisord* as an alternative:

```puppet
  mcollective_choria::federation_broker{"london":
    instances => 2,
    stats_base_port => 8000,
    srv_domain => "ldn.example.net",
    service_provider => "systemd"
  }
```

If you need to manage the *supervisord* class elsewhere or with Class Parameters, set the *manage_supervisord* parameter to *false*.

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
