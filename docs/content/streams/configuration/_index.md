+++
title = "Configuration"
weight = 10
+++

Recent versions of Choria all have Choria Streams ready to be used, though I suggest starting to use it only since version
*0.23.0* and newer.

## Choria Broker

If you have a clustered Choria Broker cluster you need to enable Choria Streams on all nodes in the cluster using the
Hiera data below:

```yaml
choria::broker::stream_store: /var/lib/choria/
choria::broker::advisory_retention: 30d
choria::broker::advisory_replicas: 1
choria::broker::event_retention: 30d
choria::broker::event_replicas: 1
choria::broker::machine_retention: 30d
choria::broker::machine_replicas: 1
```

When enabled Choria Streams will automatically create the following streams:

|Stream|Description|
|------|-----------|
|CHORIA_EVENTS|Archive of Choria Cloud Events that you would see using `choria tool event`|
|CHORIA_MACHINE|Cloud Events published by the Choria Autonomous Agent system, including Scout check states|
|CHORIA_STREAM_ADVISORIES|Audit logs of Streams API access|

The full reference of available configuration options can be seen using `choria tool config network.stream`,
not all settings are currently settable in Puppet.

## NATS CLI

Choria does not provide management CLI tools for Stream maintenance, however you can use the [NATS CLI](https://github.com/nats-io/natscli)
to interact with JetStream.

Once installed, configure the CLI to access choria:

```nohighlight
$ nats context add choria --select \
   --description "Choria" \
   --server nats://choria.example.net:4222/ \
   --tlscert /home/rip/.puppetlabs/etc/puppet/ssl/certs/rip.mcollective.pem \
   --tlskey /home/rip/.puppetlabs/etc/puppet/ssl/private_keys/rip.mcollective.pem \
   --tlsca /home/rip/.puppetlabs/etc/puppet/ssl/certs/ca.pem
```

After this `nats account info` should report something like this:

```nohighlight
$ nats account info
Connection Information:

               Client ID: 7186
               Client IP: 10.244.1.142
                     RTT: 57.874176ms
       Headers Supported: true
         Maximum Payload: 1.0 MiB
       Connected Cluster: CHORIA
           Connected URL: nats://choria.example.net:4222
       Connected Address: 10.244.1.140:4222
     Connected Server ID: NCRRT2R2R5O5FF3AMXO6VHKEH2ADITSCTVARZHXOKF2SISD4CUOTUWTS
   Connected Server Name: choria.example.net

JetStream Account Information:

           Memory: 0 B of 0 B
          Storage: 1.2 GiB of 0 B
          Streams: 6 of 0
        Consumers: 3 of 0
```

And the list of Streams can be seen:

```nohighlight
$ nats stream report
Obtaining Stream stats

╭───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│                                                               Stream Report                                                               │
├──────────────────────────┬─────────┬───────────┬──────────┬─────────┬──────┬─────────┬────────────────────────────────────────────────────┤
│ Stream                   │ Storage │ Consumers │ Messages │ Bytes   │ Lost │ Deleted │ Replicas                                           │
├──────────────────────────┼─────────┼───────────┼──────────┼─────────┼──────┼─────────┼────────────────────────────────────────────────────┤
│ CHORIA_EVENTS            │ File    │ 0         │ 3,288    │ 1.0 MiB │ 0    │ 0       │ broker-broker-0, broker-broker-1, broker-broker-2* │
│ CHORIA_STREAM_ADVISORIES │ File    │ 0         │ 14,356   │ 14 MiB  │ 0    │ 0       │ broker-broker-1*                                   │
│ CHORIA_MACHINE           │ File    │ 1         │ 142,753  │ 397 MiB │ 0    │ 0       │ broker-broker-0, broker-broker-1, broker-broker-2* │
╰──────────────────────────┴─────────┴───────────┴──────────┴─────────┴──────┴─────────┴────────────────────────────────────────────────────╯
```

Past that, for the moment, reference the [NATS Documentation])(https://docs.nats.io/jetstream).
