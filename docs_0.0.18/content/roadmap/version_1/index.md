+++
title = "Version 1"
weight = 420
toc = true
+++

Version 1 of these modules and plugins seems about due after good feedback from everyone.

There is a [GitHub milestone](https://github.com/ripienaar/mcollective-choria/milestone/2) with most of these captured
and will likely be more accurate

Apart from obvious things like being generally stable and reliable I specifically want these features:

 - [x] Secure easy to deploy SSL based security system that uses Puppet CA
 - [x] Secure easy to deploy SSL secured NATS connector with SRV support
 - [x] PuppetDB and PQL integration
 - [ ] Complete Playbooks to solid v1
 - [ ] The new JSON based audit plugin on by default
 - [ ] A *choria* GitHub org
 - [x] A *choria* forge namespace
 - [ ] Versioned docs
 - [ ] Logo

For playbooks:

 - [ ] k/v reader for inputs with at least consul supported but more possible
 - [ ] k/v writer task with at least consul supported but more possible
 - [x] reports, versioned format
 - [ ] versioned playbook document even though we wont have schemas yet

Stretch Goals, but probably only after 1.0.0:

 - [ ] A replacement for *mco rpc*
 - [ ] ActiveMQ that's like the NATS one - unconditional SSL, same SRV records
 - [ ] RabbitMQ that's like the NATS one - unconditional SSL, same SRV records

