+++
title = "Extending Choria"
pre = "<b>8. </b>"
weight = 80
+++

This section covers various ways of writing new logic that execute on your nodes or the client side. The sections here are advanced topics, many users will find interacting with [Choria via CLI](/docs/concepts/cli/) or [Playbooks](/docs/playbooks) sufficient. For many writing a [Puppet Task](/docs/tasks) will be much easier than the model presented here.

We feel the work presented here provide a richer more mature model that let you write robust logic that runs on your network and robust continuous or adhoc automation tools tailored to your needs.  This section will show you what we mean when we say Choria is a Framework that help you build your orchestration systems.

On the nodes the primary focus will be around _Agents_ which are API's your nodes expose and that are secured using the [Choria AAA](/docs/configuration/aaa/) (Authentication, Authorization and Auditing) system.  These agents tend to be installed on every node using Puppet modules like those we provide on the [Puppet Forge](https://forge.puppet.com/choria).  Examples of these agents are the ones Choria install out of the box for managing [Puppet](https://forge.puppet.com/choria/mcollective_agent_puppet), [Services](https://forge.puppet.com/choria/mcollective_agent_service) and [Packages](https://forge.puppet.com/choria/mcollective_agent_package).

On your _Client_ - where you run _mco ...._ commands - the interaction comes in the form of API clients, CLI clients or specialised tools like Playbooks.  You can use the APIs to write your own custom clients to manage your infrastructure in any way you choose such as custom daemons that react to outages and do remediation by interacting with the Agents on your nodes via the APIs.

A large focus of this work is in adapting the client and agent model created in The Marionette Collective and having Choria present a compatibility layer, the documentation here is largely adapted from this legacy system.
