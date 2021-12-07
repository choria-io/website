---
title: "AAA Improvements"
date: 2021-12-07T00:00:00+01:00
tags: ["security"]
draft: false
---

Choria supports a distributed authentication model as well as a centralised model using our Choria AAA Service. A Puppet user
uses the distribution method by default.

In distributed mode every client has a certificate, signs his request with it and the certificate becomes the identity.  The servers will verify using their RPC Authorization system if that certificate (id) can perform an action.

In the centralised setup each client do not have a certificate but it has a JWT token obtained from a sign-in service often using `choria login`. The JWT holds the identity, policies, permissions and more. The AAA Service signs requests using its certificate allowing clients to publish signed requests.  Effectively the signing step gets outsourced to a trusted 3rd party. Before signing a request a policy is evaluated on the AAA Service to determine if the request should be allowed.

The AAA Service [was introduced in 2019](https://choria.io/blog/post/2019/01/23/central_aaa/) and we've improved on it in 2020 by allowing [a client certificate free operation](https://choria.io/blog/post/2020/09/13/aaa_improvements/).

The Certificate Free operation was a big win, however it came at a considerable cost of requiring additional Choria Brokers to take client connections.

We made a number of improvements in Release 0.6.0, read the full entry for details.

<!--more-->
## Signing via Choria RPC

Previous versions of the AAA Server only supported listening on a HTTP(s) port for signing requests. Users had to provision Load Balancers, extra certificates and more. This never felt right to me as we really want in-region communications to be always via the Choria Broker.

Recent Choria Servers support a concept called Choria Services. A Service is essentially a Choria Agent that is called in a 1 to 1 basis rather than 1 to many.

Think of it like a standard microservice where 1 microservice serves 1 request.  The Choria Broker has full Service Mesh capabilities (failover, scale up, scale out, GSLB, observability and more), so a Choria Service benefits from all of these behaviors without additional components - just run multiple instances.

The latest AAA Service supports connecting to the Choria Broker and offering signing as a Choria Service. This means you have one less port to manage, do not need to worry about its HA or load balancers infront of it and do not need to worry about certificates for it. Just start multiple AAA Service and let the Choria Broker internal Service Mesh handle it all.

```json
{
  "basicjwt_signer": {
    "signing_certificate": "/etc/aaasvc/pki/aaa-privileged.choria.crt",
    "max_validity":"2h",
    "choria_service": true
  },
}
```

Setting `choria_service` to true enables this.

On the client side we have to set these settings:

```ini
# communicates with the service for signing
plugin.choria.security.request_signer.service = true

# enables non mTLS connections
plugin.security.client_anon_tls = true

# where to store the JWT file
plugin.choria.security.request_signer.token_file = ~/.choria/token

# where to perform the sign in, typically near your central SSO
plugin.login.aaasvc.login.url = https://aaa.choria:8080/choria/v1/login
```

This is currently only supported with the Go clients, Ruby to follow.

## Improved JWT Tokens

We are undertaking a significant effort to reduce our reliance on Certificate Authorities.  Traditionally every client and every server had a x509 certificate - the entire system operated only on a mTLS basis.

This is very secure and easy to understand, however, it has significant drawbacks. Operating large scale CA infrastructure for 100s of thousands of nodes can be a huge, and expensive, undertaking. Various enterprise CA operating models make relying on enterprise CA in a mTLS basis impossible and insecure.

These concerns don't really worry the typical Puppet User but non Puppet Choria users can struggle quite a bit with this.

Further, we need to convey additional information about clients and servers.  For example: since we have [Choria Streams](https://choria.io/docs/streams), there's a whole new level of ACL needed, not about what can be done on nodes but what can be done on the Broker.

So our new Client JWTs have a map of permissions, which when none are set true allows only classic Choria management but with additional security such as private reply channels: a JWT authenticated client cannot inspect traffic, replies, destined for other clients.

|Permission|Description|
|----------|-----------|
|`streams_admin`|Full access to the Choria Streams API|
|`streams_user`|User level access to Choria Streams, cannot make new streams but can operate on existing ones|
|`events_viewer`|Can view choria events, autonomous agent events and broker events (non system)|
|`election_user`|Can perform leader election against the standard Choria Leader Election system|
|`org_admin`|Can access all subjects and data in the broker within the organisation|
|`service`|Allowed to have a JWT that exceeds the default maximum life cycle|

In addition to this, clients now have a Ed25519 private key and they sign a [nonce](https://en.wikipedia.org/wiki/Nonce) on the broker to connect. This serves the same purpose as the private key in a mTLS setup - it confirms you are the rightful owner of the JWT you present to the server and to AAA. The public key is baked into the JWT and signed by the Login Service.  So you can only use a JWT if you have the matching Ed25519 seed (private key).

```json
{
  "userlist_authenticator": {
    "signing_key": "/etc/aaasvc/pki/aaa-privileged.choria.key",
    "validity": "1h",
    "users": [
      {
        "username": "admin",
        "password": "...",
        "broker_permissions": {
          "org_admin": true
        }
      }
    ]
  }
}
```

Here we make a `admin` user with the ability to subscribe and publish on all subjects in the users org. 

We've also extended the `choria tool jwt` to be able to render client tokens and their built in permissions and policies.

## Non mTLS on the main broker

With all the above JWT work and Ed25519 keys we now have all the moving parts in place to safely and securely accept non mTLS connections on the main Choria Brokers.

This is an opt-in, by default it's still all mTLS only.

```ini
plugin.choria.network.client_signer_cert = /etc/choria/pki/aaa-privileged.choria.crt
```

Once you configure the Broker with the x509 certificate used to issue the Client JWTs the broker will:

 * Require a client that presents a JWT signs a [nonce](https://en.wikipedia.org/wiki/Nonce)
 * Verify the JWT presented is signed using the configured certificate
 * Verify the nonce was signed using the public key that's in the JWT - confirming the client holds the Ed25519 private seed
 * Set permissions based on the permissions map in the JWT
 * Set up private reply channels
 * Prevent all other subjects from being accessed

This model co-habits with traditional mTLS connections meaning you can mix and match where needed.

The `choria login` command has been improved to create the needed Ed25519 keys, keys are rotated with every login.

## Organisations

We're slowly working towards formalising our multi tenancy in the Choria Broker. You notice above things like "on all subjects in the users org", this means that if you had multiple tenants on your Choria Infrastructure that an `admin` user in the `development` org can not also see subjects and traffic in the `production` org.

It's a bit early days, so this mainly a point of clarification, in time the Broker will provision new orgs when clients connect.

## Deprecations

We removed the Okta authentication plugin and also the NATS Streaming Server auditor. I am not aware of anyone using Okta in this use case so I decided to remove it and revisit later a better integration with something similar.

NATS Streaming Server will be end of life soon, so, we're also retiring our support for it. Choria Streams is supported for Auditing.

## Status

At present this is only supported in the Go client, the Ruby clients can support this model we just need to do the work.

This is available in 0.6.0 of the AAA Service and requires at least version 0.25.0 of the Choria Broker and Client (the next version).
