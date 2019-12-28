+++
title = "Key Concepts"
pre = "<b>2. </b>"
menu = "concepts"
weight = 20
+++

## What is it for?

### Day One

Immediately after installing Choria you can start managing Packages, Services, Puppet and gather information about files.

You do this using commands like:

```nohighlight
$ mco package status apache -W country=uk -C apache
```

Here we will get the installed Apache version on all our machines in the *UK* (if you have a Facter fact called country) that has the *apache* class applied by Puppet.  Information will be gathered in parallel on all the nodes concurrently.

You can use it to discover nodes, inspect their classification and perform single actions across your cluster.

{{% notice tip %}}
You can try this right now without installing anything using our [Vagrant Demo](https://github.com/choria-io/vagrant-demo) environment
{{% /notice %}}

### First Weeks

To manage other parts of your infrastructure, like the Puppet CA, you can install other agents from our [Puppet Forge](https://forge.puppet.com/choria). If you have any Puppet Tasks already or see any [on the forge you like](https://forge.puppet.com/modules?utf-8=%E2%9C%93&page_size=100&with_tasks=true) you can configure [Tasks](../tasks/) support and execute those over Choria without SSH and with full RBAC in a really performant manner.

You can start writing [Playbooks](../playbooks/) that allow you to combine agents, tasks, different databases, webhooks and more into complex multi step orchestrations.

You can write your own Tasks and use them via Choria or Bolt and share them with others on the forge.

### Second Month

If you have advanced orchestration needs not handled by the above options and you are very familiar with the use of Choria you can [start writing completely custom Agents and Clients](/docs/development/). These could manage almost any conceivable thing and present any user interface you like.

If you ever wanted to write small utilities that can interact with your fleet but were afraid of the daunting task of designing connectivity, ensuring its secure, figuring out addressing or even just did not want a ever growing list of daemons and listening ports. Now you have a framework that let you concentrate on just your problem while gaining a strong unified security model encompassing Authentication, Authorization and Auditing - you won't have to write 1 line of code in those areas, just write the utilities you need and let Choria handle those boring aspects.

### Further Exploration

You can start building large scale node metadata stores using our Data Adapters to do stream processing using the data from your nodes.

You can embed a choria server into completely custom software, perhaps you have microservices written in Golang.  With Choria embedded in those microservices they will gain a [management backplane](https://github.com/choria-io/go-backplane) that you can reach using the *mco rpc* CLI, the API or you can use the management backplane to extract running status from them. Perhaps you wish to build a big red button that can disable processing if ever there is a problem - with Choria embedded in microservices that's a quick *mco rpc circuit_breaker off -W country=us* away.

If you are keen on IoT you can embed Choria into your small sensor devices and create lifecycle management for your entire IoT fleet.

See our [Related Projects](/docs/concepts/related/) page for links to other projects we work on in this space.

## Introduction

Choria is an Orchestration System built using a client server model. Servers on your network will expose API's that allow you to perform actions against those servers.  These actions can be implemented using several different languages and you can build your own to extend the system.

Choria is aware of your network topology and provides a rich discovery system allowing you to address nodes classified by your configuration management system with certain characteristics, real time status of nodes or by querying asset databases.

The system is part complete end user solution via features like its rich Playbook system and part framework allowing you to extend it and build on it. You can start with basic Choria and do useful things on day one but sculpt it into the orchestration system of your dreams due to the exposed framework that provides many of the capabilities orchestration systems need.

In addition to basic interactive orchestration Choria provides a ever growing feature list for the systems builder where you can embed Choria capabilities into your software at compile time allowing for custom management backplanes to be built and in the IoT world custom life cycle and secure data exfil.

Choria has a very strong focus on use in the Enterprise. Every aspect of Choria is integrated with core AAA functionality with industry standard PKI Authentication, rich RBAC based Authorization and detailed Auditing.  Its communication medium does not require any direct-to-node connection from a user like using SSH, instead it relies on a very performant and scalable middleware system. Nodes connect out, no listening ports on your servers.

Using Choria you can start managing your packages, service and Puppet agents within this hour, write expressive playbooks using the Puppet DSL and, if you wish, in time join others who have built entire PaaS on top of Choria at their core using its framework.

The entire Choria project and associated eco system projects are licensed under the [Apache Licence, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).

Explore this section of the documents for an overview of its capabilities, there are sections dedicated to Deployment and Configuration later in this guide.
