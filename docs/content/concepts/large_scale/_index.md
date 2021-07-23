+++
title = "Large Scale Design"
weight = 208
toc = true
+++

This is a reference design for a very large scale Choria deployment. This deployment supports a job orchestration system that allows long-running, multi-site/multi-region workflows to be scheduled centrally while allowing region level isolation.

This is not a typical deployment that one would build using Puppet, but it does demonstrate the potential of the framework and how some of these concepts can be combined into truly large-scale orchestration platform.

This architecture has been deployed successfully across global organisations with 100s of Data Centers / Regions, 100s of thousands or even millions of fleet nodes, both on-prem and cloud based.

## Goals

The goal of the system is to provide a connective substrate that enables a very large enterprise workflow system to be built. 

The kind of workflow can be as simple as executing a single command on 10s of thousands of nodes in a region or as complex as a multi-stage workflow that can run for a long time, orchestration 100s or 1000s of steps.

Single workflows can impact machines in multiple regions, have sub workflows and have dependencies on completion of steps that happen in other regions. Everything built on a foundational mature security model providing strong Authentication, Authorization and Auditing.

In this document we will not dwell much on the actual workflow engine rather we show how the various components of Choria would enable such a system to be built reliably and at large scale. A conceptual design of the actual workflow system might be included later, but the actual flow engine is not a component included in Choria today. One can mentally replace this workflow component with any other component that need access to programmable infrastructure.

The kinds of workflows discussed are for tasks like data gathering, remediation, driving patch exercises or providing intel during outages additionally, regular observation of fleet state via [Choria Scout](https://master.choria.io/docs/scout/) can feed into ML models enabling work in the AI Ops space.

It's been shown that in large enterprises this system can support millions of workflow executions on a monthly basis once integrated into enterprise monitoring and other such systems.

A number of systems are required to realise such a system, a few are listed here and this document will explore how these are built using Choria technologies:

 * Ability to deliver the agent with low-touch overhead with self-managing of its lifecycle, configuration, availability and security.
 * Integration into existing Enterprise security be it Certificate Authorities, SSO or 2FA tokens from different vendors and with different standards.
 * Node introspection to discover its environment, this includes delivery of metadata gathering plugins at a large scale.
 * Fleet metadata collected and stored for every node, regionally and globally, continuously.
 * Telemetry of node availability.
 * Fast and scalable RPC subsystem to invoke actions on fleet nodes using mature backend programming languages. The RPC system should provide primitives like canaries, batching, best-effort or reliable mode communication and more.
 * Various components on fleet nodes can communicate their state, job progress and other information into the substrate without having to each manage reliable connections which would overwhelm the central components with short-lived connections.
 * Open data formats used to maximise integration in a highly heterogeneous environment, maximising return on investment.
 * Efficient processing of fleet state data using Enterprise programming languages using familiar streaming data concepts.
 * Isolated cellular design where individual cells (around 20 000 fleet nodes) can function independently.

## Overview

As mentioned the system will be cellular in design - in Choria we call each cell and Overlay. Each Overlay has a number of components:

 * [Choria Network Brokers](https://choria.io/docs/deployment/broker/) with the [Choria Streams](https://choria.io/docs/streams/) capability enabled
 * [Choria Provisioner](https://github.com/choria-io/provisioning-agent) that provisions individual fleet nodes onto the Choria network
 * Enterprise Certificate Authority integrated into *Choria Provisioner* to ensure a strong mTLS is in use in compliance with enterprise security
 * [Choria AAA Server](https://github.com/choria-io/aaasvc) providing integration with Enterprise SSO systems, pkcs11 tokens and more
 * [Choria Stream Replicator](https://github.com/choria-io/stream-replicator) allowing data from an overlay to be replicated to a region or to a central location

Generally, depending on the availability model chosen for the Network Brokers, this is around 3 to 5 nodes required to manage 20 000 fleet nodes.

![Client Server Overview](../../large-scale/global-orchestrator.png)

Here we see the various components built with multiple Overlay networks shown, highlighting the full independence of each Overlay.

The *Stream Replicator* component is used to move data between environments in an eventually consistent manner, that is, should *Overlay 1* be isolated from the network as a whole the *job infrastructure* can submit jobs locally. Job statuses, job progress records and metadata gathered during the time and so forth are all kept in the Overlay until such time as the connectivity to the rest of the world is restored. At that time the Stream replication will copy the data centrally.

This builds an environment where, even when isolated, an Overlay can manage itself, auto remediate itself, affect change based on input from external systems etc. Any actions taken while isolated will later flow to the central location for auditing and tracking purposes.

Essentially the workflow system becomes a system of many cooperating workflow systems with each Overlay being fully capable but that can cooperate to form a very large global system.

## Node Provisioning

In a typical Choria environment Puppet is used to provision a uniform Choria infrastructure integrated into the Puppet CA. This does not really work in large enterprises as the environments tend to be in all shapes, sizes and generations. There might be Baremetal managed using ancient automation scripts, VMWare systems, OpenStack systems, Kubernetes based container infrastructure and everything in between running in different environments from physical data centers to cloud to PaaS platforms.

Choria therefore supports a provisioning mode where the process of enrolling a node can be managed on a per-environment basis allowing for:

 * Custom endpoints for provisioning where needed. Optionally programmatically determined via plugins that can be compiled into Choria
 * Fully dynamic generation of configuration based on node metadata and fleet environment
 * Integration into Certificate Authorities provided by Enterprise Security - provided they meet some requirements
 * Self-healing via cycling back into provisioning on critical error
 * On-boarding of machines at a rate of thousands a minute

The component that owns this process is the *Choria Provisioner* and within *Choria Server* a special mode can be enabled by using JWT tokens.

The *Choria Provisioner* presents a *Choria Broker* managed by itself, creating a fully isolated on-boarding network. Optionally on-boarding can be done on the main Choria Brokers for the environment where un-provisioned machines are isolated from provisioned ones using the multi-tenancy features of the broker.

The general provisioning flow is as follows:

 * Choria Server starts up without a configuration
   * It checks if there is a *provisioning.jwt* in a well known location (see `choria buildinfo`)
   * The token is read, from it is taken server list, SRV domain, locations of metadata to expose and more.
   * Choria Server activates the *choria_provisioning* agent that is compiled into it, this agent is usually disabled in Puppet environments.
   * Choria Server connects to the infrastructure described in the token and wait there for provisioning. Regularly sending CloudEvents and, optionally publishing metadata regularly.

Here the *provisioning.jwt* is a token that you would generate on a per Overlay or Region basis and it would hold provisioning configuration for that region.

The *Choria Provisioner* actively listens for CloudEvents indicating a new machine just arrived in the environment ready for provisioning. Assuming sometimes these events can be missed it also actively scans, via the discovery framework, the network for nodes ready for provisioning.

 * For each discovered node
   * Retrieve *provisioning.jwt* JWT token, validates it for validity and checks its signature against a trusted certificate
   * Retrieve metadata such as facts, inventory, versions and more about the node
   * Requests the node to generate a Certificate Signing Request with specific CN, OU, O, C and L set. The private key does not leave the node.
   * Pass the inventory and CSR to an external extension point which must:
     * Get the CSR signed using environment specific means
     * Construct a node specific configuration that can be varied based on environment, metadata and more
   * Provisioner sends the signed certificate and configuration to the new fleet node
   * Fleet node restarts itself, now running it's new configuration using new PKI

We mention an external extension point, this is just a script or command that reads data on it's STDIN and writes the result to STDOUT. Any programming language can be used. Using this one can provide a single provisioner for an entire region or even globally, the provisioner can decide what Overlay to place a node in.

This is an important capability, being able to intelligently place nodes in the correct Overlay means all nodes that form part of the same customer service in a region can be in the same Overlay meaning in an isolation scenario all infrastructure for a client can still be managed.  One can also imagine that having a shared databased in use by 10s of customers it would be beneficial to place the database, and those customers in the same Overlay.

The decision about which Overlay to place a node can be made by metadata provided by the node and by querying Overlay level metadata storage to find the associations. One can also use this to balance, retire or expand the Overlays in a region by for example re-provisioning all nodes in a specific Overlay and then spreading those nodes around the remaining Overlays.

Being that the extension point is an external script any level of integration can be done that a user might want without recompiling any Choria components.

**Further Reading:** 

 * [Mass Provisioning Choria Servers](https://choria.io/blog/post/2018/08/13/server-provisioner/)
 * [Upcoming Server Provisioning Changes](https://choria.io/blog/post/2019/12/30/whats_next_for_provisioning/)
 * [Provisioning Agent Repository](https://github.com/choria-io/provisioning-agent) (warning: out of date README, improvements imminent)

We have some TODO items on the Provisioner:

 * Make it highly available so that a standby Provisioner in a data center can take over when the primary one fails, now possible using Choria Streams
 * Perhaps extend the information that can be embedded in the CSR
 * In addition to certificates allow a token to be placed on the node to access Choria Broker multi tenancy model

## Certificate Authorities

As described above the Provisioning Server can be used to integrate Choria into enterprise Certificate Authorities. There's a lot of detail in this one point though, and the CA has to have certain properties:

 * The CA **MUST** issue certificates with a SAN set. Go will not allow certificates without SANs to be used
 * Fleet node certificates must have the node FQDN in them
 * User or role certificates need to have specific SAN set or have common names matching Choria identity patterns like *rip.mcollective*. Specifics on this point varies by environment or use, and is an area in desperate need of improvement.
 * The certificates and keys **MUST** be in a x509 PEM format
 * The certificates **MUST** be RSA based - a restriction we are working on to relax
 * We can set custom CN, OU, O, C and L fields, additional fields would need to be set by the CA or we need to extend provisioning, we want to support requesting specific SANs.
 * The certificates should in general allow for server and client use, it might be acceptable for fleet nodes to only have client mode restrictions. Testing required.
 * All certificates must allow signing.
 * We support auto enrollment for Fleet nodes but not yet for brokers and other components unless they are in Kubernetes managed by cert-manager. Those must have long validity certificates issued by the CA and presented to the node using configuration management.
 * The CA must have some form of API so it can be called at from the provisioning script
 * We recreate certificates between provisioning attempts. That is a node *c1.example.net* might present new CSRs signed by new keys several times during its life, we would not know what the sequence of the past certificates was so the CA or integration must handle revocation if needed
 * The API has to be fast, if there is a network with 60 000 fleet nodes it will not do to provision them 5/second. We need to be able to issue 1000s per minute.
 * On connections from Overlays to central (see the stream replicator), the Overlay must have a certificate that would give it access to that central environment and must have access to the CA chain.

It might be desirable to instead obtain an intermediate CA from the Enterprise CA and issue certificates locally using something like *cfssl* or integration into Vault.
