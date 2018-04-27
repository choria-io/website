+++
title = "Key Concepts"
weight = 2
icon = "<b>2. </b>"
+++

## Introduction

Choria is an Orchestration System built using a client server model. Servers on your network will expose API's that allow you to perform actions against those servers.  These actions can be implemented using several different languages and you can build your own to extend the system.

Choria is aware of your network topology and provides a rich discovery system allowing you to address nodes classified by your configuration management system with certain characteristics, real time status of nodes or by querying asset databases.

The system is part complete end user solution via features like its rich Playbook system and part framework allowing you to extend it and build on it. You can start with basic Choria and do useful things on day one but sculpt it into the orchestration system of your dreams due to the exposed framework that provides many of the capabilities orchestration systems need.

In addition to basic interactive orchestration Choria provides a ever growing feature list for the systems builder where you can embed Choria capabilities into your software at compile time allowing for custom management backplanes to be built and in the IoT world custom life cycle and secure data exfil.

Choria has a very strong focus on use in the Enterprise. Every aspect of Choria is integrated with core AAA functionality with industry standard PKI Authentication, rich RBAC based Authorization and detailed Auditing.  It's communication medium does not require any direct-to-node connection from a user like using SSH instead it relies on a very performant and scalable middleware system. Nodes connect out, no listening ports on your servers.

Using Choria you can start managing your packages, service and Puppet agents within this hour, write expressive playbooks using the Puppet DSL and, if you wish, in time join others who have built entire PaaS on top of Choria at their core using its framework.

The entire Choria project and associated eco system projects are licensed under the [Apache Licence, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).

Explore this section of the documents for an overview of its capabilities, there are sections dedicated to Deployment and Configuration later in this guide.