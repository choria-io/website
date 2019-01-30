---
title: "Centralised AAA"
date: 2019-01-23T13:28:00+01:00
tags: ["security"]
draft: false
---

Choria is a very loosely coupled system with no central controller and in fact no shared infrastructure other than a middleware that is completely "dumb".  What this means is there is no per request processing anywhere centrally other than just to shift the packets.  No inventory databases, user databases or other shared infrastructure to scale or maintain - though several integration options exist should you choose to do so.

There are many reasons for this - in a large scale environment there are always things broken and automation systems should do their best to keep working even in the face of uncertainty. This design extends from the servers, middleware all the way to the client code. The loosely coupled design ensures that what can be managed will be managed.

This is generally fine and works within my design parameters and goals. For the client though in enterprise environments this is problematic:

 * Enterprises are heavily invested in SSO and entitlement based flows for permissions
 * Enterprises and regulated environments have strong requirements for auditing to centralized systems
 * Certificate management for individual users is a nearly impossible hurdle to scale

So today I would like to present a new extension point that allow you to fully centralize AAA for the Choria CLI.

<!--more-->
## Some terminology

Lets just define what we mean with AAA, it's a common term generally meaning Authentication, Authorization and Auditing:

* Authentication - validating that you are who you claim to be, generally using something you have or know - passwords, yubikeys, certificates and keys.
* Authorization - inspecting your role and permissions and allowing you to do only what you are allowed to do on the network
* Auditing - logging or recording what you have done

## The current state

Choria has a very strong story already on all of the 3 A's in its decentralised deployment. It's really important that whatever centralised strategy we choose cohabit with the loosely coupled AAA that exist today.

So what I will present allow you to pick centralised auth for your users - but certificate based auth for your automated systems, cron jobs etc - and freely mix and match as you see fit.

The Authorization and Auditing components should work identically regardless of how the user is Authenticated.

### Authentication

You have a certificate and key signed by a Certificate Authority, the certificate must comply to a naming pattern and be signed by the CA that the nodes trust.  This way they know something else have declared you are who you are.  Once your identity is confirmed this is passed onto the Authorization and Auditing frameworks.

### Authorization

Given that it knows who you are we can express what you are allowed to do, the widely used (and deployed by default) [Action Policy](https://github.com/choria-plugins/action-policy) system provides a strong Authorization story

### Auditing

Since it knows who you are and what you are allowed to do, audit logs are written using a JSON log like below, this is logged on every node.

```json
{
  "timestamp": "2019-01-22T16:28:53.672386+0000",
  "request_id": "f374d0d5a79258858a92025efe0ecd03",
  "request_time": 1548174533,
  "caller": "choria=rip.mcollective",
  "sender": "dev1.example.net",
  "agent": "rpcutil",
  "action": "ping",
  "data": {
    "process_results": true
  }
}
```

## Centralisation Overview

Choria has for a long time supported the concept of a *Privileged Certificate*, this is a certificate that can assert the caller is someone other than the common name of the certificate.  Usually this is not allowed, if you hold a certificate for John then you are John and cannot suddenly claim to be Jill.

Privileged certificates can set any callerid they wish, this way a central signing service can sign the request on your behalf and it can encode in the request that it signed it on your behalf. This way the central signing service can create unique data that the Authorization and Auditing frameworks need while not requiring a certificate for every user.  Your audit logs and action policy rules would show the unique per user caller id (set by the signer).

I have a few goals with this work, its quite ambitious but I've been working on this for a long time. Even back in 2010 I've demonstrated 2FA based centralised authentication that's identical to this for the MCollective CLI. This work formalises that and make it generally available:

 * support any login schema possible
 * be scalable even to many data centers - even when those data centers cannot talk to the central SSO service
 * be able to support things like 2FA and all modern strategies without the need for per user client certificates
 * to avoid copying certificates between nodes to identify a user

The centralisation we support will be a 2 step process:

 * Login to your central system and obtain a token
 * Request a central signer signs your request which would include the token

We split the login and signing stage since I believe these are typically going to be run in different places.

The Login service with access to your SSO will probably run centrally while signing has to happen near as possible to the client - in every DC.  The use of JWT makes the process entirely stateless, the signing service never have to speak to the central system once the JWT was created.

### Logging in

I anticipate the interface for logging into the service is just not something I can make generic enough, there are 100s of systems in use by companies - OAuth, SAML, OpenID, static user lists, Radius, LDAP, the list is endless and we could never support all possible cases.

Further you might implement the token in many different ways - [JWT](https://jwt.io) is a common pattern but many others exist.

So we do not provide the login part of this at all, we anticipate you'd write a *application* plugin, something like `mco login` that prompts for a user and password. It would communicate with your login service that would issue a token.  The token can then be stored in a file or the environment.

So this is entirely free form, we do not get involved in this. Below you can see the basic flow of a login request, here we use Okta to store our users and groups and derive a JWT token based on group membership.

![Login Flow](login.svg)

This is the flow the tool I will release supports when using Okta.

### Signing Requests

When it comes to signing requests we need to support a standard since the CLI code needs to do this, in practice you could create your own plugin to do anything at all but I'd encourage sticking with the standard.

Choria will attempt to speak to a webservice described by the  OpenAPI specification - [json](https://choria.io/schemas/choria/signer/v1/service.json), [yaml](https://choria.io/schemas/choria/signer/v1/service.yaml).  Here you receive a standard [Choria Request](https://choria.io/schemas/choria/protocol/v1/request.json) and the token from the Login.

The service will then decide if it wants to sign the request and if it does it uses the *privileged certificate* so do so.  It returns a standard [Choria Secure Request](https://choria.io/schemas/choria/protocol/v1/secure_request.json)

![Signing Flow](signing.svg)

Note there's an auditing step here, you can do whatever you want, I publish a message to my NATS Stream where the Choria Stream Replicator sends the data from the regional DC to a central DC to be ingested into my central system.  This way we always know every request that was signed anywhere in the fleet.

## Configuring Choria

This feature is available in version 0.13.1 and newer of the Choria Client, configuring it is pretty easy:

```ini
plugin.choria.security.request_signer.url = http://signer.example.net:8080/choria/v1/sign
plugin.choria.security.request_signer.token_environment = CHORIA_TOKEN
plugin.choria.security.request_signer.force = 1
```

Or if you wanted to fetch the token from a file:

```ini
choria.security.request_signer.token_file = ~/.choria_token
```

## Upcoming work

This is pretty new, I'd love to hear feedback from the community on how this looks, I will soon release a login and signer that support Okta and statically configured users as well as NATS based auditing. Today this code lives in [choria-io/aaasvc](https://github.com/choria-io/aaasvc) - it's an early take on this that will evolve to a production ready service.

This is a very important space, I hope to work with other users who have other generic authenticator needs so we can round this out to a fully realised central auth system for Choria.
