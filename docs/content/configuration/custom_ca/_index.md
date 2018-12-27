+++
title = "Custom Certificate Authority"
toc = true
weight = 270
+++

While Choria is configured by default to use the Puppet CA the system does support custom Certificate Authorities including intermediaries.  You can use any software to produce these certificates as long as they make compliant x509 certificates.

This section will guide you through the creation of a layered CA setup for a Choria network using the [Cloudflare's PKI toolkit](https://cfssl.org/).

{{% notice tip %}}
This is support from Choria Server 0.9.0 and the the mcollective-choria module version 0.12.1
{{% /notice %}}

## Overview

This guide will cover the following:

 * Creating a *Example Root CA* that is used to sign a per datacentre intermediate CA
 * Creating a *London CA* and a *New York CA* that issues certificates in each datacentre
 * Create user certificates in each datacentre and configure Choria CLI
 * Configure individual servers in each datacentre

What this guide will not cover:

 * How PKI works. [This guide](https://smallstep.com/blog/everything-pki.html) is very good.
 * How to get the certificates on every node, this step is essentially going to be unique per site, we cannot realistically cover this for everyone. The [Choria Server Provisioner](https://github.com/choria-io/provisioning-agent) can enroll nodes in any CA with an API
 * Certificate revocation and renewal
 * How to use CFSSL in detail or its deployment best practices
  * For documentation on the CFSSL project, and how to run it in a client/server fashion, please visit the [Cloudlare CFSSL documentation repo](https://github.com/cloudflare/cfssl/tree/master/doc)

## Requirements

 * You need to be running Choria Server 0.9.0 or later, the Ruby `mcollectived` does not support this.

## Root Certificate Authority

In PKI the Root CA is your main CA that you generally keep offline on something like a USB stick or encrypted somewhere.  You use it to generate new Intermediate Certificate Authorities - in our example each data center is a Intermediate CA.  It is very important that you protect this secret.  If it's compromised,  your entire chain of trust will be broken.  An intermediate CA will help reduce the scope of the damage.

The data center then deploys a Certificate Authority bundle that declares the chain of trust and states which Certificate Authorities are to be trusted by a particular node.

Here we will create our Root Certificate Authority using CFSSL, this involves a few steps:

* Initialize the Root CA
* Set the signing policy of the intermediate CAs

### Initialize the CA

CFSSL works by reading JSON files that represents the certificate requests etc, lets create one describing our Root CA:

Create `csr_root.json`:

```json
{
    "CN": "Example Root CA",
        "key": {
            "algo": "ecdsa",
            "size": 256
        },
    "names": [
        {
            "C": "MT",
            "L": "Siggiewi",
            "O": "Example",
            "OU": "Example Root Certificate Authority"
        }
    ],
    "ca": {
        "expiry": "262800h"
    }
}
```

The Root CA has a validity of 30 years and you can see the various properties.

We can now initiate our Root Authority:

```
cfssl gencert -initca csr_root.json | cfssljson -bare root_ca
```

You should see files `root_ca.csr`(certificate signing request), `root_ca.pem`(public key) and `root_ca-key.pem`(private key).  The private key for your CA should be stored in a safe location, preferably offline, and encrypted.

### Signing Policy

We need to state what the policy this CA will follow when signing new Certificate Authorities, this we will do using the file `intermediate_ca_signing.json`:

```json
{
    "signing": {
        "default": {
            "usages": ["digital signature","cert sign","crl sign","signing"],
            "expiry": "262800h",
            "ca_constraint": {
                "is_ca": true,
                "max_path_len":0,
                "max_path_len_zero": true
            }
        }
    }
}
```

This means Certificate Authorities being created by this CA will also have a long life and will be able to sign certs on their own.

{{% notice tip %}}
Of particular note is the _usages_ field - these will specify Extended Key Usage policies on the request which downstream consumers can use to validate the uses of the certificate, should they be presented with it.  Choria enforces these policies by default.

The *ca_constraint* key is to prevent the intermediate certificate from being able to create additional certificate authorities underneath it (or, if the value is set to non-zero, that many levels of CAs underneath it).  For the purposes of this demonstration, we are leaving it at zero.
{{% /notice %}}

## London Intermediate CA

Each DC will need to generate a CSR that will be processed by the Root CA.  This signing process creates a chain of trust from the client and server certificates, combined with the intermediate CA, to the root CA.  As mentioned with the root CA private key, the intermediate CA private keys should be protected to the policies of your organization has for any important PKI infrastructure(ie offline if possible, encrypted if not).

We'll do the following:

 * Create the Key, CSR etc of the CA
 * Sign the CSR at the Root Ca
 * Configure the CA signing policies
 * Create the local bundle

I will show how to do this for London, you can just repeat this process for other DCs.

### Create the London CA

Lets create `london_ca_csr.json`:

```json
{
    "CN": "London CA",
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "UK",
            "L": "London",
            "O": "Example",
            "OU": "London Certificate Authority"
        }
    ],
    "ca": {
        "expiry": "87600h"
    }
}
```

Here we are describing the signing request for the London CA that has a validity of 10 years.  Lets create the key and x509 csr:

```
cfssl gencert -initca london_ca_csr.json | cfssljson -bare london_ca
```

This will create the following files:

|File|Description|
|----|-----------|
|london_ca.csr|The signing request that the Root will sign|
|london_ca.pem|The unsigned intermediate so it's useless, you can discard this one|
|london_ca-key.pem|The private key for your CA, do not lose this or share it|

### Sign the London CA using the Root CA

Now you need to transport this `london_ca.csr` to your Root CA, do not worry there is nothing important in here you do not need to keep it private or super secure.

Do this on the Root CA:

```
cfssl sign -ca root_ca.pem -ca-key root_ca-key.pem -config intermediate_ca_signing.json london_ca.csr | cfssljson -bare london_ca
```

The main output here will be a file called `london_ca.pem`, this is the London CA signed by your Root CA.  Copy this file back to your London CA.

### Configure signing policy for London

Create a file `server-signing.json` that has the following:

```json
{
    "signing": {
        "profiles": {
            "default": {
                "usages": ["signing", "key encipherment", "server auth"],
                "expiry": "43800h"
            }
        }
    }
}
```

This states that when signing a request from a particular server it should be able to do server related things - but not for example make new CAs - and that the signed certificate will be valid for 5 years.

### Chained certificates

When creating infrastructure with TLS and x509 certificates, you have several options.  At minimum, all servers will need to have a copy of the `root_ca.pem` file.  Servers should have a copy of their local intermediate CA (in this example, `london_ca.pem`) file in their CA bundle (as shown below), and clients can optionally present their own local intermediate CAs as part of their negotiation - this is particularly useful in cross DC requests where you might not have distributed the CA bundle universally, or have rotated the remote intermediate CA.

In all cases, the certificate file on disk should contain the canonical certificate of the _server_ or _client_ first, _then_ the intermediate CA(s).  It should not contain the root CA file.

The CA root bundle ordering does not matter, but by convention the root CA should be present at the top.

### Create the London Bundle

You should grab a copy of the root `root_ca.pem` and the `london_ca.pem` to create the bundle:

```
cat root_ca.pem london_ca.pem > bundle.pem
```

This file will be distributed to all the clients and servers within the _London_ DC.

## Enrolling a node

We'll issue certificates for every node, this entails:

 * Creating the private key and CSR
 * Signing it in the data center CA
 * Configuring the Choria Server

{{% notice tip %}}
The [Choria Server Provisioner](https://github.com/choria-io/provisioning-agent) can automate this enrollment of nodes in a Certificate Authority, using it is not for everyone but if you are a big complex site it might be of interest to you.
{{% /notice %}}

#### Creating the CSR

Every node will need a certificate matching its _fqdn_, lets create the CSR for the server, first the JSON request in `/etc/choria/ssl/csr.json`:

```json
{
    "CN": "server1.example.net",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "UK",
            "L": "London",
            "O": "Example Inc",
            "OU": "Operations"
        }
    ]
}
```

Now we generate our private key and the x509 format CSR:

```bash
cfssl genkey csr.json | cfssljson -bare server1.example.net
```

### Signing it using the London CA

The previous step would create your private key and a few others, you should transport the `server1.example.net.csr` to the `London CA` host and sign it there:

```bash
cfssl sign -ca london_ca.pem -ca-key london_ca-key.pem -config server-signing.json server1.example.net.csr | cfssljson -bare server1.example.net
```

This will produce a `server1.example.net.pem` that you should copy back to the node into `/etc/choria/ssl`

You should also grab the `bundle.pem` and place this in `/etc/choria/ssl/ca.pem`

### Configure Choria

By default Choria Server will try to use certificates in the official locations Puppet Agent puts them, we need to switch that to manual paths in `/etc/choria/server.conf`

```json
plugin.security.provider = file
plugin.security.file.certificate = /etc/choria/ssl/server1.example.net.pem
plugin.security.file.key = /etc/choria/ssl/server1.example.net-key.pem
plugin.security.file.ca = /etc/choria/ssl/ca.pem
plugin.security.file.cache = /etc/choria/ssl/cache
```

## User Certificates

Just as nodes need to be able to identify their identities so should every distinct user who access Choria.  To enroll a user we need more or less the same things as the node needed above:

 * Creating the private key and CSR
 * Signing it in the data center CA
 * Configuring the Choria Client

### Creating the private key and CSR

We'll create `/home/alice/.choria.d/ssl` and place all the files in there for the user `alice`.

Create the JSON CSR `/home/alice/.choria.d/ssl/csr.json`:

```json
{
    "CN": "alice.mcollective",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "UK",
            "L": "London",
            "O": "Example Inc",
            "OU": "Operations"
        }
    ]
}
```

Now we generate our private key and the x509 format CSR:

```bash
cfssl genkey csr.json | cfssljson -bare certificate
```

### Sign the CSR in the London DC

The previous step would create your private key and a few others, you should transport the `certificate.csr` to the `London CA` host and sign it there:

```
cfssl sign -ca london_ca.pem -ca-key london_ca-key.pem -config server-signing.json certificate.csr | cfssljson -bare certificate
```

This will produce a `certificate.pem` that you should copy back to the node into `/home/alice/.choria.d/ssl/`

You should also grab the `bundle.pem` and place this in `/home/alice/.choria.d/ssl/ca.pem`

### Validating chain of trust

The `openssl` command can by used to validate the chain of trust.  For the above example:

```bash
openssl verify -CAfile bundle.pem certificate.pem
certificate.pem: OK
```

If you are using intermediate certificates in _client_ or _server_ certificates(where the client/server certificate has the intermediate certificates present _after_ the public identity certificate), you'll want to specify the `-untrusted` flag:

```bash
openssl verify -CAfile bundle.pem -untrusted certificate.pem certificate.pem
certificate.pem: OK
```

### Configure Choria

Assuming you are enabling this pattern for all users on the machine you can edit  `/etc/puppetlabs/mcollective/client.cfg`:

```ini
plugin.security.provider = file
plugin.security.file.certificate = ~/.choria.d/ssl/certificate.pem
plugin.security.file.key = ~/.choria.d/ssl/certificate-key.pem
plugin.security.file.ca = ~/.choria.d/ssl/ca.pem
```

If done correctly the `mco choria show_config` command will show the paths to the certificate files and report a valid setup.

## Conclusion

At this point you should be able to use `mco` cli as normal, nodes will validate your certificates are signed by the CA and matches who you claim to be.
