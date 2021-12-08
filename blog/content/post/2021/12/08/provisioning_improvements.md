---
title: "Provisioning Improvements"
date: 2021-12-08T00:00:00+01:00
tags: ["operations", "provisioning"]
draft: false
---

The typical Choria Deployment method is to use Puppet to provisioning everything on the managed nodes.  This works fine for those users, however on [Large Scale](https://choria.io/docs/concepts/large_scale/) this just does not really work. 

Large Enterprises have a vastly varied infrastructure, and you simply do not find Puppet in use across all tiers. We therefore support provisioning Choria in a way that's entirely configuration management free.

Essentially this is the "IoT Light-bulb" mode, you start a Choria Server and in short period of time its figured out how to provision itself, connected to the provisioning infrastructure and were on-boarded.

The Choria Provisioner can provision thousands of nodes a minute, is highly available and extendible and can integrate with enterprise CAs.

[In August](https://choria.io/blog/post/2021/08/13/secure_and_ha_provisioning/) we blogged about some enhancements to make this processes better, today we follow up with further improvements.  Read on for full details.

<!--more-->
## Server JWT Tokens

We are undertaking a significant effort to reduce our reliance on Certificate Authorities. Traditionally every client and every server had a x509 certificate - the entire system operated only on a mTLS basis.

We recently added improved [Client JWT Tokens](https://choria.io/blog/post/2021/12/07/aaa_improvements/) that can be obtained using the AAA Service. In this release we bring the same advantages to Servers.

When provisioned using Choria Provisioner a Server will now create a private Ed25519 seed and send the Public key to the Provisioner. The Provisioner will verify that the Server holds the correct seed by expecting the new Server to sign a nonce.

The Provisioner will create a unique JWT for the node in question, sign it and send it back once signed using an RSA certificate. The JWT is sent at the same time as the rest of the node configuration, poliicies and more.

Embedded in the JWT is the Identity of the server, a new set of Permissions, the Organization and Sub Collectives a server belong to.

Since we now know when a Server connects to the broker even what Sub-collectives it belongs to we create a set of tightly controlled permissions for this server, this is the first time Server access to the broker is being locked down.

 * Servers cannot get direct requests for other servers
 * Servers can only subscribe to requests in the approve list of sub collectives
 * Servers can not read replies from other servers
 * By default, Streams, Submission and hosting Services are all disabled

The JWT token has a few embedded Permissions:

|Permission|Description|
|----------|-----------|
|`submission`|Allows the server to publish to Choria Streams using [Choria Submission](https://choria.io/docs/streams/submission/)|
|`streams`|Allow client access to Choria Streams - for example to read KV values|
|`service_host`|Can host Choria Services|

Additionally, if Servers are configured to publish Registration to specific subjects this can be allowed on a per subject based in the JWT.

The `choria tool jwt` CLI tool can render these tokens.

## Non mTLS connections

In the past we had no supported way for servers to escape the mTLS requirements.  Given these additions detailed above we can now allow servers to connect without mTLS.

This is an opt-in, by default it’s still all mTLS only.

```ini
plugin.choria.network.server_signer_cert = /etc/choria/pki/node-jwt-signer.crt
```

Once you configure the Broker with the x509 certificate used to issue the Server JWTs the broker will:

 * Require a server that presents a JWT signs a nonce
 * Verify the JWT presented is signed using the configured certificate
 * Verify the nonce was signed using the public key that’s in the JWT - confirming the server holds the Ed25519 private seed
 * Set permissions based on the permissions map in the JWT
 * Set up private direct-request channels
 * Prevent all other subjects from being accessed

This model co-habits with traditional mTLS connections meaning you can mix and match where needed.

## Status

This is available in 0.13.0 of the Choria Provisioner and requires at least version 0.25.0 of the Choria Broker and Server (the next version).
