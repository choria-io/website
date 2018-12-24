+++
title = "Terminology"
weight = 204
toc = true
icon = "<b>3. </b>"
+++

As with most modern complex systems Choria has it's own verbiage, this page attempts to define the terminology we use when discussing or writing about Choria.

## Action

[Agents](#agent) expose tasks, we call these tasks actions. Each agent like a exim queue management agent might expose many tasks like *mailq*, *rm*, *retry* etc. These are al actions provided by an agent.

## Agent

Code that captures a specific API a server wish to expose to Choria Clients. Example agents are *package*, *service*, *puppet* and ones you write your own.  Agents expose [Actions](#action) to clients.

They can be written in Ruby via the MCollective Compatibility Framework or in Golang and soon other options will be made.

## Auditing

A log kept by the RPC framework of all actions performed on a server

## Authentication

The process of securely identifying a user using PKI

## Authorization

Sometimes known as RBAC, a list of rules that are used to decide if a request should be authorized.  Builds on Authentication to apply the right rules to the right user.

## Broker

A piece of software that facilitates communication between disconnected entities. The Choria Broker incorporates the Choria Network Broker, Choria Federation Broker, Choria Data Adapter

See also [Middleware](#middleware)

## Choria Data Adapters

A micro-services framework hosted within the Choria Broker process that receives data from subsystems like Registration and republishes it into other frameworks like Stream Processing systems.

Delivered as part of the *choria* single binary.

## Choria Federation Broker

A Choria protocol aware router and intelligent gateway that connect several independent Collectives together into 1.  Also performs co-processing for Clients significantly reducing the work they have to do.

Delivered as part of the *choria* single binary.

## Choria Server

A *Server* written in Golang. It can run standalone and replace *mcollectived* but can also be embedded into other Golang projects at compile time.

Delivered as part of the *choria* single binary.

## Choria Network Broker

A managed instance of the [NATS.io](https://nats.io) Server, capable of serving 50 000 or more connection on a single compute node.

Delivered as part of the *choria* single binary.

## Client

Software that produce commands for servers to process, typically this would be a computer with the client package installed and someone using the commands like *mco package* to interact with Agents.

## Connector

A plugin of the type *MCollective::Connector* that handles the communication with the Choria Broker.

## Collective

A combination of Servers, Nodes and Middleware all operating in the same Namespace.

Multiple collectives can be built sharing the same Middleware but kept separate.

## Facts

Discreet bits of information about your nodes. Examples could be the domain name, country, role, operating system release etc. These are often gathered and exposed by Configuration Management systems like Puppet.

## Federation

Federation in distributed systems is typically software that combines isolated systems into one larger system.  Choria supports Federation allowing you to build a combined Federated Collective that have as it's members many isolated Collectives.

See [Federations of Collectives](../../federation)

## Marionette Collective / MCollective

A Orchestration System written by the author of Choria and sold to Puppet Inc. This system was included in Puppet since 2009 and sunset in late 2018.

Choria builds on many of the ideas, modernizes a lot of the concepts and provide a compatibility framework for MCollective agents while looking towards the future.

## Middleware

A [publish subscribe](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) based broker used to communicate between Clients and Servers.

See also [Broker](#broker)

## Node

The Computer or Operating System that the Server runs on. A compute node or physical machine.

## Plugin

Code that lives inside the server and takes on roles like security, connection handling, agents and so forth. See [Wikipedia](https://en.wikipedia.org/wiki/Plug-in_(computing))

## Registration

A process where the Choria Server publish on a regular basis data from a node, this could be metadata or in a IoT setting data like temperature, humidity and pressure.

## Server

The *mcollectived* or *choria server* daemon, an app server for hosting Agents and managing the connection to your Middleware.

Deployed to every node you wish to manage.

## Simple RPC

A Remote Procedure Call system built on top of MCollective.  Choria provides a compatibility layer for this so that Agents and CLI Applications users wrote in the past will continue to function.

## Subcollective

A server can belong to many Collectives. A Subcollective is a Collective that only a subset of a full collectives nodes belong to.

Subcolllectives are used to partition networks and to control broadcast domains in high traffic networks.

