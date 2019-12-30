---
title: "Upcoming Server Provisioning Changes"
date: 2019-12-30T09:00:00+01:00
tags: ["operations", "provisioning"]
draft: false
---

I've previously blogged about a niche system that enables [Mass Provisioning Choria Servers](https://choria.io/blog/post/2018/08/13/server-provisioner/), it is quite scary and tend to be specific to large customers so it's off by default and require customer specific builds to enable.

I've had some time with this system and it's proven very reliable and flexible, I've had many 100s of thousands of nodes under management of this provisioner and can kick 10s of thousands of nodes into provisioning state and it will happily do thousands a minute.

The concept is sound and the next obvious step is to make it available to FOSS users in our FOSS builds. To get there is a long road one that I think will take us toward Kubernetes deployed Choria Broker, CA and Choria Provisioners and eventually a Puppet free deployment scenario.

It's a long road and step one is about looking at how we can safely enable provisioning in the open source builds.  Read on for the details of what was enabled in Choria Server 0.13.0 that was released on 12th of December.

<!--more-->

On the surface nothings really changing in the general model introduced in the [Mass Provisioning Choria Servers](https://choria.io/blog/post/2018/08/13/server-provisioner/) post, what is changing is that you can now create a JWT token that embeds in it information that could previously only be compiled into custom binaries.

This token doesn't really get validated much by the individual nodes (they do not yet have any certificates etc when this is started), but the central provisioner will validate this token and only provision nodes that have tokens signed by a trusted certificate.

## Generating Tokens

You'll need a x509 certificate pair that is used to sign and validate the token, it doesn't really matter if these are from a CA you trust or not, the pair is whats needed - you can of course issue it from your CA.

This creates `prov-key.pem` and `prov-cert.pem`:

```nohightlight
$ openssl req -x509 -newkey rsa:4096 \
  -keyout prov-key.pem -out prov-cert.pem -days 1825
```

We have a new CLI tool `choria tool jwt` used to issue these tokens:

```nohighlight
$ choria tool jwt provisioning.jwt prov-key.pem \
  --insecure \ 
  --token toomanysecrets \
  --urls prov1.example.net:4222,prov2.example.net:4222 \
  --default \
  --registration /etc/facts.json \
  --facts /etc/facts.json
```

You can also use SRV records instead of static URLs using `--srv provision.example.net`, this will result in SRV lookups against `_choria-provisioner.tcp.provision.example.net` to locate the Choria Brokers.

|Flag|Description|
|----|-----------|
|--insecure|Since the Choria Server has no certificates or CA yet - thats part of provisioning - it's common to run these without TLS, which is fine since no private keys traverse the network|
|--token|Post provisioning the provisioning agent will be enabled, but you can only interact with it with this token, it's just a long string.|
|--urls|A comma separate list of Choria Broker URLs|
|--srv|Instead of hard coding the URLs you can give a SRV domain here|
|--default|Enables provisioning unless specifically disabled in the config, good for missing/corrupt config handling|
|--registration|The Choria Server will regularly publish the contents of this file to the network|
|--facts|The Choria Server will use these facts during provisioning for discovery and will expose that to the provisioner|

Place this file in `/etc/choria/provisioning.jwt` and you can confirm with `buildinfo` if it worked:

```nohighlight
$ choria buildinfo
...
Server Settings:
            Provisioning Default: true
                Provisioning TLS: false
    Provisioning Target Resolver: Default
      Default Provisioning Agent: true
              Provisioning Token: set
            Provisioning Brokers: prov1.example.net:4222, prov2.example.net:4222
           Provisioning JWT file: /etc/choria/provisioning.jwt
...
```

From this point onward everything is pretty much as discussed previously with the addition now that the Choria Server Provisioner will have the public key that you created above and confirm that the JWT token is still valid and that it was signed by the private key in question.

You can also view a token if you have the public key:

```nohighlight
% choria tool jwt provisioning.jwt prov-cert.pem
JWT Token provisioning.jwt

                         Token: set
                        Secure: false
                          URLS: prov1.example.net:4222, prov2.example.net:4222
       Provisioning by default: true
              Standard Claims: {
                                 "iat": 1577723210,
                                 "iss": "choria cli",
                                 "nbf": 1577723210,
                                 "sub": "choria_provisioning"
                               }
```

## Backwards Compatible

If you have previously created your own provisioning agent, custom builds or used the `buildsettings.yaml` to compile these options into your Choria Server this will continue to function. But this gives a smaller touch way to enable this.

## Next Steps

As I said this is step one in a long road, short term we'll:

 * Perhaps rename some of the keys in the JWT to be more spec compliant
 * Give the `choria/choria` module the ability to not manage `server.conf` so you can install plugins etc with Puppet but handle core config with provisioner
 * Hand in hand with the above change `choria/choria` will allow placing the JWT on the nodes

Longer term:

* Create a set of Helm charts that will set up:
  * A Certificate Authority for Choria
  * Choria Server Provisioner to enroll new nodes into the CA and give them configs
  * Choria AAA Service to give users access to the network managed in this manner
  * Dashboards for Provisioning, AAA and Lifecycle event Tallys
  * A shell for interacting with the fleet using the `mco login` flow
  * Move some of the Ruby agents into the binary
  * Deliver all the rest of the Ruby agents as part of the RubyGem

Once we have these ticked off basically you'll have a Puppet free setup.