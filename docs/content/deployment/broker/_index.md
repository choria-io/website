+++
title = "Choria Network Broker"
weight = 110
toc = true
+++

Choria has its own broker that is based on the excellent [NATS.io](https://nats.io/) broker.  It's very fast, lightweight, is easy to configure and monitor.

  * TLS out of the box using your Puppet Agent certificates, accepting only nodes signed by Puppet CA
  * Handles 50 000 MCollective connections per node - can be increased with a custom build
  * Supports clustering on a LAN using a Full Mesh
  * Expose extensive Prometheus.io compatible metrics
  * Distributed for RedHat, Debian and Ubuntu
  * Includes advanced features like [Federation](../../federation) and [Protocol Adapters](../../adapters).

## Prerequisites

 * The Broker must be managed by Puppet and have certs signed by your CA
 * The Brokers must run RedHat 7 or 8, recent Ubuntu or recent Debian
 * You need to ensure port `4222` is reachable from all your Puppet nodes to all the Choria Broker servers
 * You need to ensure that in a clustered environment port `4223` is reachable between all the Choria Broker servers

## Single or multiple nodes

The decision to run multiple nodes is about availability and scale.  As mentioned the Choria Network Brokers can easily handle large numbers of nodes on a single broker, if this is your first deployment there is no reason right now to think about a cluster of brokers.  As you'll see configuring a cluster is very easy and easily done later.

If you choose to do 1 only keep it simple and install it on your Puppet Server.  This removes the need to configure DNS (the next section) and gets you going ready to explore the possibilities quickly, you can easily later add servers.

A Broker with 10 000 connected nodes consume around 300MB RSS.

## Federated or Single Cluster

As the Choria Network Broker only support a Full Mesh architecture it is not a good idea to run one single large globally distributed Choria Network Broker cluster.  Choria supports Federating many small collectives together to facilitate geographic deployments. Federation would also make sense if you are in one location but have many thousands of nodes.

As before if you're getting started you should focus on deploying a single location and familiarising yourself with Choria.  Should you then choose to go ahead review the [Federation](../../federation) section of the documentation.  Deploying Federation is easy but you need to understand a bit more architecturally and it has additional monitoring utilities which are best covered seperately.

## Package Repos

Packages are hosted with [packagecloud](https://packagecloud.io/choria/release), you can mirror the repositories and set up your own repository configuration or use the packagecloud ones.

To configure the Choria Repositories use the following class:

```puppet
class{"choria":
    manage_package_repo => true
}
```

## Single node

If you just want to run a single NATS server I suggest putting this on the same machine as your Puppet Server which would by default be resolvable as _puppet_.  This means you do not need to configure anything in Choria as that's the default it assumes.

```puppet
node "puppet.example.net" {
  class{"choria::broker":
    network_broker => true
  }
}
```

## Cluster of Choria Brokers

You can create a cluster of brokers, pick 3 or 5 machines and include the module on them all listing the entire cluster certnames. If you do a cluster you must configure Choria via [DNS or manually](../dns/).

```puppet
node "nats1.example.net" {
  class{"choria::broker":
    network_peers => [
      "nats://choria1.example.net:4223",
      "nats://choria2.example.net:4223",
      "nats://choria3.example.net:4223",
      "nats://choria4.example.net:4223",
      "nats://choria5.example.net:4223"
    ],
  }
}
```

## Large deploys

A note about large deploys such as when you have many thousands of nodes connected to a single NATS server.  NATS has no problem handling this but your kernel might consider the large amount of reconnects after you restarted NATS as a DDoS attempt and throttle the SYN packets.

{{% notice tip %}}
You can use the [thias/sysctl](https://forge.puppet.com/thias/sysctl) module to manage kernel parameters
{{% /notice %}}

You can overcome this by adjusting these sysctl settings:

```yaml
net.core.somaxconn: 4092
net.ipv4.tcp_max_syn_backlog: 8192
```
