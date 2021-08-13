---
title: "Provisioning HA and Security"
date: 2021-08-13T00:00:00+01:00
tags: ["operations", "provisioning", "streams"]
draft: false
---

The *Choria Provisioner* is a niche component that can onboard Choria Servers into a Choria environment without needing
Puppet or other CM. I often refer to this as *light-bulb mode*, ie. a IoT device style on-boarding rather than traditional CM.

I've written in the past about this in [Mass Provisioning Choria Servers](https://choria.io/blog/post/2018/08/13/server-provisioner/)
for background.

Today I want to talk about upcoming changes to significantly improve this process from a security and reliability
perspective and talk a bit about what is next.

Read on for more details.

<!--more-->
Till now Provisioning mode Servers would connect in the clear to a dedicated broker (often hosted in the Provisioner itself),
where a single Provisioner would handle on-boarding these nodes.

This presented a few problems - the most obvious is that tokens and such would be traveling over the network in the clear -
but also that the entire Provisioning flow was a big single point of failure as it did not support running multiple Provisioners.

We set out to address this in recent work, as of the next release of the various components the provisioning would happen
against the same Brokers as where your fleet runs. You probably already invest in making those highly available - so we want
to use that investment.

Additionally, using [Choria Streams](https://choria.io/docs/streams), we support making the provisioner highly available by 
performing Leader Election against it.

![Client Server Overview](/docs/large-scale/unverified-tls-provisioning.png)

The above diagram shows the basic flow from a TCP and NATS perspective going forward, this requires some explanation.

## Accounts

The Choria Broker supports multi tenancy, 2 tenants on the same Broker do not share any information at all between them
by default. Sharing is possible but no sharing by default.

This means in the above diagram that any connection associated with the `Provisioning Account` is entirely isolated from
those in the `Choria Account`.

This is a key concept to understand, the Broker authentication and authorization will ensure that any server presenting
itself for provisioning will be placed in the `Provisioning Account`. It will also ensure that the Provisioner is running
within that account.

The end result is an entirely separate network within the same shared infrastructure. Think of it like a VLAN.

## Provision Tokens

Provisioning is off by default, back in the day you would have had to recompile Choria yourself to gain the ability
to auto provision it.

No more, since a year or so you can place a signed JWT file on the server that instruct it to support entering provisioning.

```nohighlight
$ choria tool jwt provisioning.jwt provisioner-key.pem --urls nats://choria.example.net:4222 --default
```

Here we create a `provisioning.jwt` signed using `provisioner-key.pem`.

When placed in `/etc/choria/provisioning.jwt`, and without a valid configuration, Choria Server will enter Provisioning.

```nohighlight
$ choria buildinfo
...
Server Settings:
            Provisioning Default: true
                Provisioning TLS: true
    Provisioning Target Resolver: Default
      Default Provisioning Agent: true
              Provisioning Token: *****
            Provisioning Brokers: nats://choria.example.net:4222
           Provisioning JWT file: /etc/choria/provisioning.jwt
...
```

The Broker will only accept servers presenting this token and if it's been signed using the `provisioner-key.pem`.

## TLS Modes

Finally, we can talk about TLS in the above diagram. Any Choria Server or Client must be over a fully mutually validated
TLS connections aka mTLS. Without recompiling everything there are no exceptions.

The problem with Provisioning is that at the point of Provisioning the Server does not yet have a valid TLS Certificate
signed by the CA - delivering this is a major task of the Choria Provisioner.

So we now support a mode where we accept a mix of Verified and Unverified connections when Provisioning is configured.
By default, we are still only ever full mTLS. This is something you have to enable.

Unverified connections are subject to a number of mitigating checks:

 * Connections coming in on unverified connections MUST present a provision.jwt that MUST validate using the public certificate the broker has in its configuration
 * Only servers with a `provision.jwt` is allowed to connect, those servers have very strict permissions. They cannot communicate with any other unprovisioned servers and may only start a specific few agents
 * The Choria Provisioner MUST connect over verified TLS and must present a password matching one the broker has in its configuration. Only the Choria Provisioner can make requests to unprovisioned nodes.
 * The entire provisioning system is isolated within the broker in an Account - meaning there is no Choria communications between provisioned and unprovisioned nodes
 * Life-cycle events are transported into the Choria account and into Streams to facilitate a central audit stream

This means a) any connection that's unverified will be isolated from the fleet b) the only component that can provision 
node MUST use fully verified mTLS.

Together this means we have a secure on-boarding process, safe from snooping and token leaks.

Enabling this mode is quite easy on the broker:

```go
plugin.choria.network.provisioning.signer_cert = /etc/choria/provision-signer.pem
plugin.choria.network.provisioning.client_password = provS3cret
```

Here we say any incoming Provisioning mode server must present a JWT that will be verified using `/etc/choria/provisioning-signer.pem`
and we say the Choria Provisioner will log in with that password.  Both have to be set to enable any Unverified TLS connection.

## High Availability

In the past the Choria Provisioner could not be run more than once per network. Since introducing Choria Streams we
can now do Leader Election against that component.

So, if the Choria Brokers support Streams, you can set the Provisioner to do Leader Election and start as many as you want.

We are not very aggressive with the Election Campaigns you will see 10 or 20 seconds of outage during a Failover but that's
completely acceptable for this use case.

## Next Steps and Conclusion

Provisioning is quite niche, but, it's something that I see will take a bigger role in the future, I'm investing in making this
robust and safe before expanding its role.

Today it only support TLS enrolment and Configuration, we will expand this:

 * Support configuration Policies - action policy and Rego based
 * Support deploying one or more Choria Autonomous Agents

