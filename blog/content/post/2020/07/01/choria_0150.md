---
title: "Choria Server 0.15.0"
date: 2020-07-01T09:00:00+01:00
tags: ["releases"]
draft: false
---

We have quite a significant release of the Choria Server today that lays ground work for an upcoming Choria monitoring
pipeline.

Read the full post for the details.

<!--more-->
## Choria Server 0.15.0

This release includes 2 major sets of improvements along with a list of day to day improvements and changes to provisioning

Thanks to Romain Tartière for his contributions.

### Choria Scout Infrastructure

Choria Scout is infrastructure needed to build monitoring event pipelines using Choria. In this release the initial
support infrastructure is landing which includes the ability to run Nagios compatible plugins regularly on the fleet
nodes and perform auto remediation.

Statuses and events are published as CloudEvents format and can be stored in our Streaming solution, integrated with
3rd parties and more.

We also support initial integration with Prometheus Node Exporter to surface some of this data to Prometheus.

For Puppet users adding checks is easy:

```puppet
choria::scout_check{"check_acme":
  plugin            => "/usr/lib64/nagios/plugins/check_procs",
  arguments         => "-C acmed -c 1:1",
  remediate_command => "service acmed restart",
  interval          => "2m"
}
```

This will continuously check the service using the standard Nagios plugin and remediate should it fail, events are
published to the network and all this is exposed to Prometheus and other systems.

The Puppet module releases will follow in the next few days.

You'll find a number of dashboards related to Choria already on the Grafana repository with more to come.
 
We'll publish follow up blog posts about Scout.
 
### Preview Streaming Server

Part of my work with [NATS.io](https://nats.io) is the new NATS JetStream product that will replace NATS Streaming
Server as a streaming data solution.

Choria already support JetStream in various parts - data adapters and more - in this release we allow you to host a
JetStream instance within the Choria Broker.  Our Helm charts makes it easy to connect a JetStream instance to the 
network (more on that below).

This is a preview feature and pretty easy to setup:

```ini
# data retention in /data/jetstream
plugin.choria.network.stream.store = /data/jetstream

# autonomous agent related events kept for a day
# in a stream called CHORIA_MACHINE
plugin.choria.network.stream.machine_retention = 24h

# lifecycle events kept for a day in a stream called CHORIA_EVENTS
plugin.choria.network.stream.event_retention = 24h

# connects as a leafnode to a service hosting a choria broker cluster
plugin.choria.network.peer_port = 0
plugin.choria.network.leafnode_remotes = broker
plugin.choria.network.leafnode_remote.broker.url = nats://broker-ss:7422
``` 

For full details about the capabilities this provide, how to interact with it and set up more streams please see the
[Technology Preview](https://github.com/nats-io/jetstream#readme) documentation.

In time we will use this feature for a few internal optimisation like giving the Ruby clients a buffer when accessing
very large networks where Ruby fails to keep up.

This will also be used extensively in Choria Scout.

### Kubernetes Support

We are starting to work on getting Choria to play well with Kubernetes and be deployable via Helm. To get there we had
to improve a few things in how Choria behaves when deployed in a Kubernetes pod.

We're also taking a look at the CA needs and have thus far integrated Choria into [cert-manager](https://cert-manager.io),
this let components such as the Broker automatically obtain certificates from the CA without per component configuration.

The Provisioning subsystem also had some work done and it too supports cert-manager making it much easier to provision
fleets of IoT or other embedded uses for Kubernetes users.

We are publishing docker containers for the various components. These like the Helm charts are a work in progress and
feedback would be appreciated.

Speaking of Helm, we have a [chart repository](https://github.com/choria-io/helm) allowing many of the shared components
to be deployed, again, this is a work in progress and the full stack is deployable but documentation is lacking while
we get these up to scratch for our internal use.

```nohighlight
$ kubectl -n choria get pod
NAME                            READY   STATUS    RESTARTS   AGE
aaasvc-678778dff6-jncdg         1/1     Running   0          14d
aaasvc-678778dff6-x8xqj         1/1     Running   0          14d
broker-0                        1/1     Running   0          5d1h
broker-1                        1/1     Running   0          5d
broker-2                        1/1     Running   0          5d
broker-stream-0                 1/1     Running   0          2d22h
provisioner-68db79f745-xc8mm    1/1     Running   0          16d
tally-server-588c4c549c-srwjj   1/1     Running   0          16d
``` 

### Changelog

 * Support preparing for shutdown by closing connections and emiting shutdown events when embedded
 * Support NATS JetStream Streaming Server in Choria Broker
 * Support arm5 and 7 Debian packages
 * Support Nagios compatible plugins in the new `nagios` autonomous agent watcher                 
 * Server instances embedded in other software can now be shutdown using `Shutdown()`             
 * Track nodes expired by maintenance in the tally helper                                         
 * Improve FQDN resolution when running in a kubernetes pod                                       
 * Allow the public name of the network broker to be configured                                   
 * Support cert-manager.io as security provider                                                   
 * Correctly handle provisioning by SRV domain                                                    
 * Allow provisioning brokers to have user/password authentication                                
 * Perform backoffs between reconnects to the network broker                                      
 * Cosmetic improvements to windows packages                                                      
