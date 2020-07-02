---
title: "Scout Components"
date: 2020-07-03T09:00:00+01:00
tags: ["releases", "scout"]
draft: true
---

Yesterday I introduced a new Choria component called Scout which helps you build scalable monitoring pipelines. Today,
we'll look a bit at what makes a Scout install and how it is built.

In a follow up post I'll dive a bit into Autonomous Agents - an infrequently used but very powerful building block
found in Choria.

## Individual Nodes

On the individual node you'll have a Choria Server and whatever check plugins you need to health check your node, container,
server or pod.

### Choria Server

Choria Server is the component that runs on every node. For users of Choria as from [choria.io](https://choria.io) this is 
a pre-packaged daemon that can be configured using Puppet or using Choria Provisioner. Every user of this stack will get
the Scout features.

Additionally, for Scout we'll look to build a purpose specific distribution of Choria that:

 * Will not be able to host arbitrary RPC endpoints
 * Can start without any configuration, discover its provisioning host and enroll into the CA and network
 * Exposes Autonomous Agent state over an RPC API with extensive RBAC
 * Embeds [Goss](https://github.com/aelsabbahy/goss) and allow regular validations
 * Can easily be run inside a side car observing Kubernetes pods
 * Hosts the central component which enables Provisioning and communication
 * Is a single binary

In this setup TLS is not optional, and the system supports auto enrolling into a Certificate Authority 
like [Cert Manager](https://cert-manager.io/) hosted on Kubernetes.
 
### Choria Autonomous Agent

The [Autonomous Agent](https://master.choria.io/docs/autoagents/) is a hosted *Finite State Machine* (FSM) that runs
entirely on the Choria nodes without required coordination from any central point.

Autonomous Agents are similar to a lightweight Kubernetes Operator that is hosted on the nodes. I've used it to build 
IoT solutions and continuous management systems like container management. 

For monitoring the main complex part is a State Machine that cycles between OK/WARNING/CRITCAL/UNKNOWN states and executes
a plugin on a schedule. This state processing and check execution is provided by this component - a robust hosted FSM. 

Autonomous Agents let you hook watchers into particular states - so for example we can hook a `exec` watcher when in `CRITICAL`  
state it attempts to remediate the problem.

For Scout, we added a `nagios` watcher that understands the 4 code exitcode structure of Nagios Plugins. We'll add a watcher
to run `Goss` checks too.

To add new checks, a Puppet defined type `choria::scout_check` can easily add new checks. One can also add them using any
tool that can write YAML files.

In time, it will support obtaining it's check configuration from the network without requiring configuration management.

We'll look in detail at this in a follow-up post.

### Force Checks and Maintenance 

While Autonomous Agents run without interaction from third parties they do have a API that they expose. Using this API one
can enumerate what Autonomous Agents are running and can also ask them to perform a specific transition.

We support `FORCE_CHECK`, `MAINTENANCE` and `RESUME` transitions - as well as the standard `OK`, `CRITICAL`, `WARNING` and
`UNKNOWN`. Using these, tooling can be built to suspend checks on a node or force them to immediately run a check or 
transition to a specific state.

The API is accessible over Choria RPC and is subject to the full AAA controls that Choria support. Including authentication
against systems like Okta, Enterprise SSO and PKCS 11 tokens.

### CloudEvents

The Autonomous Agent subsystem emits [CloudEvents](https://cloudevents.io/) whenever a machine transitions between states 
and can also regularly send updates with the state of a Watcher.

CloudEvents is a CNCF project that has support in many third party systems - almost every Public cloud support ingesting them
in some form, and many open system support them - KNative, Serverless Event Gateway, OpenFaaS and many more.

For Scout, we publish CloudEvents about transitions and about specific checks. This means we'll be able to easily 
integrate our monitoring pipeline into other processing pipelines, workflow managers like Triggermesh and more.

These events can be consumed into NATS JetStream for archival and processing using Stream Processing tools. We'll probably
provide an event gateway that routes them to HTTP destinations or onto systems like AWS SQS.

These CloudEvents are the only communications the fleet does to the outside world and are quite lightweight. If we send
every check we should be able to scale to 10s of thousands of nodes. By sending only failing checks - and perhaps an 
infrequent update of current state - we'll be able to scale to 100s of thousands of failing checks on exceptionally large
fleet of nodes.

### Prometheus Node Exporter

The `nagios` Watcher can optionally write extensive state about checks into the Prometheus Node Exporter from where you can
collect this data and build dashboards using Prometheus and Grafana. This naturally supports alert routing using Prometheus' 
Alertmanager. Scout dashboards are listed on Grafana plugin repository.

This is an optional component, and the same data is available as CloudEvents. By observing the event stream a cluster-wide 
state view can be constructed. Choria has a component called `tally` that might be expanded to maintain a state of a managed 
network and expose that over an API.

## Central Components
### Choria Broker

Choria Broker is simply a managed [NATS](https://nats.io) instance that integrates well into Choria configuration and
sets up ACLs and such to comply with our design.

It supports enrolling into a [cert-manager](https://cert-manager.io/) CA at start and can be trivially hosted in Kubernetes.

There are some other components that go with it to facilitate node enrolment, installing all this is quite easy using
Helm on Kubernetes or Puppet for traditional environments.

One can also run this entirely serverless by using [Synadia NGS](https://synadia.com/ngs).

### Prometheus

[Prometheus](https://prometheus.io/) and [Grafana](https://grafana.com/) is a popular combination for receiving metrics
and building alerting pipelines. Prometheus can send alerts to Pager Duty, OpsGenie and many other notification services.

As before, using Prometheus is optional. If you can consume CloudEvents and react to them in some fashion - perhaps on
Azure Event Grid, Serverless API Gateway, KNative, OpenFaaS et cetera - you can construct your own pipeline for triggering
alerts and workflows.

### JetStream

[JetStream](https://github.com/nats-io/jetstream#readme) is a Streaming Messaging platform that is currently in Tech 
Preview by [NATS](https://nats.io) and we already support routing the CloudEvents into it for archival and long term 
storage.

From there one can consume them historically or in real-time to route the events to other locations. It supports Event
Sourcing patterns and a growing list of integrations and is easily configured using Cloud Native technologies like
Terraform.

## Conclusion

As I stated in the introduction post Scout will have batteries included but be a flexible framework and share its state
freely and in open formats.

CloudEvents format is being adopted by many in the industry and Scout supports that at every level, which means you're not
tied into any specific solution and your time is invested in something open and easy to integrate.
