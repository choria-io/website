---
title: "Choria AAA Improvements"
date: 2020-09-13T09:00:00+01:00
tags: ["security"]
draft: false
---

Choria introduced a [Centralized AAA](https://choria.io/blog/post/2019/01/23/central_aaa/) model in 2019
that alleviate the need for managing certificates of every user and allow you to integrate Choria into 
your enterprise identity providers for Authentication, Auditing and Authorization.

For controlled environments this model is a huge boom, but there was one annoying thing - the need to still
issue a TLS certificate to communicate with Choria Brokers. In this mode, these certificates do not form part 
of the security model of Choria but was nonetheless required to exist, you could share them but that was 
frowned upon.

In our next release we will introduce a new broker type that significantly simplifies the AAA security
model by allowing clients holding no certificates to interact, safely, with Choria networks.

<!--more-->
## Client Leafnode

NATS 2.0 - the underlying technology that enable Choria Broker - supports a technology called Leafnodes
that allow us to extend a NATS network and change the security model for those connections.

For the upcoming Choria release we utilise Leafnodes to create a broker that is setup specifically for
the purpose of supporting clients who are using the AAA Service.

![AAA Leafnode](aaa_leafnode.png)

In the above diagram the existing Choria Brokers and Choria Servers remain as ever within the fully verified
mTLS secured Choria environment.

We connect a Leafnode to the existing Choria Brokers which also uses mTLS to verify itself into the 
Choria network.

This Leafnode though is configured to accept anonymous TLS connections - it does not require any certificate
to be presented by the client and does no verification.

Usually this would, correctly, raise a number of concerns such as giving unverified clients access to the
network from where they can passively observe requests and replies. It also opens the network up to MitM
attacks.

To mitigate these concerns a number of restrictions are in place:

 * The broker will deny any Choria Server from connecting
 * The broker will not allow Clusters to be formed
 * The broker will not be able to join any Super Cluster
 * The broker will not allow a client without a valid AAA Service JWT token to connect
 * The broker will not allow anyone to subscribe to Choria Lifecycle events

Further concerns exist, inherently these clients are only secured by a JWT - something that might not be
as securely stored as a private key - so someone holding such a JWT might be able to eavesdrop on the requests
and replies of other clients thus gaining valuable intel.

NATS does not have a concept of private subject, so we use the AAA Service JWT token to determine the Choria
identity of the client that connects. Using this identity we establish a private namespace where replies from
the fleet back to the client will be sent, this reply subject is only accessible to the individual caller id
as issued by the AAA Service.

## Configuration

Client configuration follows the normal AAA based client setup but needs the `plugin.security.client_anon_tls`
setting set.

```ini
loglevel = warn

plugin.security.client_anon_tls = true

plugin.choria.security.request_signer.token_file = ~/.choria.token
plugin.choria.security.request_signer.url = https://caaa.example.net/choria/v1/sign
plugin.login.aaasvc.login.url = https://caaa.example.net/choria/v1/login

libdir = /opt/puppetlabs/mcollective/plugins
```

The only new thing here is the `client_anon_tls` setting which will arrange things so no valid TLS configuration
is required and TLS connections will be anonymous.

Broker configuration requires access to the Public Certificate that issues the Choria JWT tokens to verify
validity of the client JWTs

```ini
loglevel = info

# Instructs the broker to not verify TLS
plugin.choria.network.client_anon_tls = true

# The path to the AAA Service public certificate
plugin.choria.security.request_signing_certificate = /etc/choria/aaa/signer-cert.pem

# Certificates issued by the CA managing the Choria network
# used for full mTLS connectivity to the Choria network
plugin.security.provider=file
plugin.security.file.certificate = /etc/choria/leafnode/cert.pem
plugin.security.file.key = /etc/choria/leafnode/key.pem
plugin.security.file.ca = /etc/choria/leafnode/ca.pem

# The Leafnode connection to the Choria Network
plugin.choria.network.leafnode_remotes = choria
plugin.choria.network.leafnode_remote.choria.url = nats://choria.example.net:7422

# Enables clients
plugin.choria.broker_network = true
plugin.choria.network.client_port = 4222
```

## Conclusion

This represents a huge leap in the usability of the Choria network when managed by the AAA Service.

These leafnodes are also really good at creating satelite access to the Choria network, I run one in 
Malta which connects to my network in Europe and the US. This allows me to optimise my TLS setup time
and shaves off a good 50+ ms from fast requests.

This is an exciting addition, we'll be doing some follow up work in the new code created for this to
facilitate multi tenancy for a strongly secure version of our existing sub collective features.
