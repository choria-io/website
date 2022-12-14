+++
title = "Security"
weight = 207
toc = true
+++

Clients and Servers connect with each other using the Choria Broker (See [Architecture](../architecture/)). Connections
to the Broker must be authenticated. Choria supports 2 major methods of authenticating to the broker.

 * x509 based mTLS creating a closed network where only entities with issued certificates can connect
 * JWT Tokens with ed25519 signed connection [Cryptographic nonce](https://en.wikipedia.org/wiki/Cryptographic_nonce) over TLS

The first is the default supported method for your typical Puppet user. The Puppet CA provides a Certificate Authority
and, we have extensive integration with it.

The second is more advanced and is typically used at [Large Scale](../large_scale) in combination with the Choria Provisioner and
Choria AAA Service. You would choose this if issuing 10s or 100s of thousands of certificates is prohibitively slow or if
integration with enterprise Certificate Authorities prove to be onerous.

## x509 Mutual TLS

### Overview

This is the most typical deployment strategy and one any Puppet user is familiar with. The Puppet CA is used to issue
certificates and only those holding keys and certificates signed by the Puppet CA can connect to the broker. For the Puppet
user there are no settings to change, all defaults align with Puppet defaults, it just works.

This creates a private network where access is controlled, initially, at the CA level. Past that the identity found in
the signed certificate is used to convey caller id or server identity and various controls are added around that.  These
Caller IDs are used in RBAC, auditing and more.

We also support enrolling in a Kubernetes Cert-Manager based Certificate Authority.  If you run a broker using our Helm
charts that broker will get its certificates at start time from Cert-Manager.

Finally, if you do not work in an environment that has Puppet you can use the Choria Provisioner to enroll new Choria Servers
to the network, the Choria Provisioner supports a standard CSR based approach for enrolling nodes and can integrate with any
CA that has an API. In Kubernetes the provisioner can use Cert-Manager to issue certificates for nodes not in Kubernetes.

### When to choose

The majority of users should choose this method, it's the default, has few dependencies and plays well with Puppet. It's
very secure and tested over best part of a decade.

If you have a CA that you would rather use you can configure Choria with specific paths to certificates and keys. It would
then use those same as it does the Puppet ones.

If you have non-Puppet nodes, a mix of Puppet and non-Puppet nodes, if you pay licensing based on certificate counts or
if you find the CA is operated in a way incompatible with mTLS (frequent CA cert rotations) then look at the other option.

If you wish to adopt Choria Streams and use it for other purposes you might find you need tighter control of access to
these features, we cannot encode these permission flags in certificates so, for that, you need the JWT based model.

### Benefits

 * Deep Choria integration and a known model, client simply run `choria enroll` and their admin signs a certificate on the CA
 * No additional work needed on servers, they re-use the same certificate that Puppet uses on that node
 * Easy to understand model very similar to HTTPS mTLS
 * No additional dependencies other than what you already have
 * Optionally supports Choria Provisioner for enrolling servers
 * Optionally supports Choria AAA Server for centralising client access
 * Non-Puppet CAs supported by custom configuration of paths to certificates, keys etc and deployment using non Puppet methods
 * Non-Puppet CAs supported with node auto provisioning using Choria Provisioner
 * Clients without individual certificates supported using Choria AAA Server

### Drawbacks

 * Certificate Authorities might not be available or be inherently incompatible with mTLS due to local operation modes
 * Enrolling 100s of thousands of servers into a CA can be extremely costly due to licensing
 * Managing expiring certificates very difficult
 * Choria Broker is unaware of the type of connection and does not set strict permissions, very little access control to Streams and other features
 * Certificates are per-client, meaning if many nodes are used to access the fleet you have to copy keys and certificates around

### About Certnames

Given that the main intention here is to re-use Puppet CA we need to be a bit careful that someone who gets their hands on
any Puppet signed certificate cannot communicate with the fleet as a client.

For this purpose we make our client certificates using a pattern like `name.surname.mcollective`. The server has a configured
allowed list `plugin.choria.security.certname_whitelist` that is a list of patterned certificates must validate against in order
to be considered a valid client. This way Puppet operators can spot Choria certificates easily and configure auto signing policies
accordingly.

### Configuration

For the typical Puppet user there is nothing to configure, at start Choria will get the `ssldir` from Puppet and use the certificates
found there.

#### Custom Certificates

Custom paths to certificates can be configured as below:

```nohighlight
plugin.security.provider = file
plugin.security.file.certificate = /path/to/cert.pem
plugin.security.file.key = /path/to/key.pem
plugin.security.file.ca = /path/to/ca.pem
plugin.security.file.cache = /path/to/cache/dir # only needed for servers
```

Here we set the security subsystem to `file` over the default `puppet` and specify paths to our files.

#### Reference

For details about settings please consult the `choria tool config` tool, here a list of related settings, and some short details

| Setting                                     | Description                                                                                                                                           | Default                                             |
|---------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| `plugin.security.provider`                  | The security provider to use, defaults to, one of `puppet`, `file`, `pkcs11`, `certmanager`                                                           | `puppet`                                            |
| `plugin.security.file.ca`                   | Path to the CA chain when provider is `file`                                                                                                          |                                                     |
| `plugin.security.file.certificate`          | Path to the public certificate when provider is `file`                                                                                                |                                                     |
| `plugin.security.file.key`                  | Path to the private key when provider is `file`                                                                                                       |                                                     |
| `plugin.security.file.cache`                | Path to the certificate cache, public certificates for clients are stored there and new certificate that is different from cached one will be rejects |                                                     |
| `plugin.security.always_overwrite_cache`    | Do not prevent changing client certs from being denied                                                                                                | `false`                                             |
| `plugin.choria.security.certname_whitelist` | List of certificate patterns that are allowed to interact with servers                                                                                | `\.mcollective$`, `\.choria$`                       |
| `plugin.choria.security.privileged_users`   | List of certificate patterns that are allowed to sign requests on behalf of others                                                                    | `\.privileged.mcollective$`, `\.privileged.choria$` |
| `plugin.security.cipher_suites`             | List of allowed cipher suites                                                                                                                         | Go defaults                                         |
| `plugin.security.ecc_curves`                | List of allowed ECC Curves                                                                                                                            | Go defaults                                         |

## Organizations using JWT tokens

### Overview

This method does not rely on mTLS, instead we use anonymous TLS, Signed JWT tokens and ED25519 key pairs to provide the following:

 * TLS for encrypting traffic in transit
 * JWT tokens signed by a trusted component to create a closed network of allowed connection, controlled by a central authority
 * ED25519 key-pair to uniquely identify a connection as holding the private key used to request the JWT token
 * Rich set of role based permissions and security Policies embedded in JWT token

This deployment method combines Choria Broker, [Choria Provisioner](https://choria.io/provisioner) and 
[Choria AAA Service](https://choria.io/aaasvc) to create a Configuration Management free deployment method but can also be combined
with a Puppet based deployment method to configure plugins and more.

This method can be deployed using mTLS should it be desired for added transport safety.

### When to choose

{{% notice secondary "Version Hint" code-branch %}}
This requires Choria 0.27.0 which is due for release in early 2023
{{% /notice %}}

If your enterprise Certificate Authority is operated in a way that makes using mTLS difficult then this is your only option.

You might find that you are adopting Choria Streams heavily but want to ensure you have tighter control over access to its features
then the JWT model will allow you to set flags on a per-connection basis enabling features.

In regulated environments or ones dealing with sensitive information you might need to completely isolate clients from each other
ensuring that one client cannot possibly see replies sent to other users.  The JWT model ensure a private communications channel
is setup that is private to every client.

If you do not use Puppet this method can enable a complete Configuration Management free deployment model.

### Benefits

 * Does not require a Certificate Authority with API or the ability to issue certificates on demand
 * Supports environments where Certificate Authorities are rotated or rebuilt frequently - impossible to use mTLS
 * Permission set controlling strict access to sub collectives, Streams features and more
 * Private communication channels with isolation between clients for privacy and security of reply data
 * Short-lived sessions keys and tokens, expired hourly for user connections. Special permission flag needed to extend to longer
 * Easier to integrate with 3rd party SSO systems
 * Removes the complex need for managing policy on individual servers by, optionally, embedding policy in JWT tokens
 * Easy to integrate with enterprise single sign on and IAM
 * Compatible with running the shared infrastructure on Kubernetes
 * No single points of failure with all components supporting Highly Available deployment methods
 * Key rotation and integration with systems like Hashicorp Vault is possible

### Drawbacks

 * Not supported by our Ruby clients (planned)
 * Harder to configure with more dependencies
 * Increased key management for long-lived services
 * More centralised, meaning more dependencies on components that can fail or become unavailable
 * A new model that might still have teething problems
 * Not integrated with the Puppet modules - generally not needed for Puppet users

### Configuration

Configuring is shown for AAA Service, Provisioner, Broker and Client. I will not show the entire setup, just the specific parts
related to this kind of deployment - for full details reference the documentation for those tools.

#### Broker

The broker will perform validation of connections from Provisioning Servers, Provisioned Servers and Clients. In all cases
the JWTs are RSA signed using a privat key, we configure the public key of all 3 components in the broker to validate the
connections accordingly.

The broker will verify servers who are being provisioned using their JWT, a server must present a valid JWT signed by a known
certificate and, they will then be placed into the provisioning organisation to isolate them from provisioned nodes:

This is the Public RSA key that signed the `provisioner.jwt` that goes on nodes.

```ini
plugin.choria.network.provisioning.signer_cert = /etc/choria/broker/provisioning-jwt-signer.pem
```

While provisioning nodes the Provisioner will create a new JWT and sign that JWT using it's signing key. The broker will only
allow servers signed by that RSA key to connect, this should be a different key-pair than the one that signed `provisioning.jwt`:

```ini
plugin.choria.network.provisioning.signer_cert = /etc/choria/broker/provisioner-node-signer.pem
```

Clients who are authenticating get a signed JWT, the JWT should be RSA signed and the broker needs the public key:

```ini
plugin.choria.network.client_signer_cert = /etc/choria/broker/client-jwt-signer.pem
```

#### Provisioner

Provisioner takes care of on-boarding new nodes, it will verify the `provisioning.jwt` and then generate a configuration
and new Server JWT that will be presented to the broker.

During provisioning the server will create a ED25519 session key, the public key will be embedded in the JWT being generated
at this stage.

On connection the server will sign a NONCE that the Choria Broker will verify as being made by the corresponding private key, this
way we do not use server JWTs as bearer tokens instead we fully verify that the connecting server holds the JWT but also the private
key that was used to request the server JWT.

Below the important bits from the `choria-provisioner.yaml`:

```yaml
# public key used to sign the provisioning.jwt
jwt_verify_cert: /etc/choria-provisioner/provisioning-jwt-signer.pem

# private key used to sign server JWTs
jwt_signing_key: /etc/choria-provisioner/node-signer-key.pem

helper: /etc/choria-provisioner/helper.rb

features:
    jwt: true # verifies the provisioner.jwt
    ed25519: true # enabled non tls mode
```

In the `helper.rb` we can now set some additional properties that will configure the server JWT:

```ruby
reply["server_claims"] = {
  # token expire after a yead
  "exp" => Time.now.utc.to_i + (60*60*24*365),
  "permissions" => {
    # can be a Choria Submission node
    "submission" => true,
    # can use KV buckets and Governors
    "streams" => true
  }
}

reply["configuration"].merge!({
  "plugin.security.server_anon_tls" => "1",
  # other settings not shown
})
```

First we set claims to place in the Server JWT and we tell the server to use non mTLS connection.

The permissions supported are, all default to false:

| Permission     | Description                                                                             |
|----------------|-----------------------------------------------------------------------------------------|
| `submission`   | Allow the Server to enable [Message Submit](https://choria.io/docs/streams/submission/) |
| `streams`      | Allow the Server to access Choria Streams for KV, Governors and more                    |
| `governor`     | Allow the Server to use Governors - also require `streams`                              |
| `service_host` | Allow the Server to host Services such as Registry                                      |

#### AAA Service

For the AAA Service the main change here is in the Authenticator.  The Authenticator is optional and could be something
user provided to integrate into the enterprise SSO.

You might configure your users like this:

```json
[
  {
    "username": "puppetadmin",
    "password": "$2y$05$c4b/0WZ5WJ3nhSZPN9m8keCUPlCYtNOTkqU4fDNEPCUy1C9Pfqn2e",
    "acls": [
      "puppet.*"
    ],
     "broker_permissions": {
        "events_viewer": true
     }
  }
]
```

Here we have a user that can manage Puppet related actions and he has the `events_viewer` permission set, below a table of
all permissions.  All default to `false`.

| Permission      | Description                                                                                                                |
|-----------------|----------------------------------------------------------------------------------------------------------------------------|
| `streams_admin` | Can perform CRUD operations on Streamd and Consumers                                                                       |
| `streams_user`  | Can use Streams, request their information, create Consumers but not create new Streams                                    |
| `events_viewer` | Can use `choria tool event` to view fleet lifecycle events                                                                 |
| `election_user` | Can access the standard Choria [Leader Election](https://choria.io/docs/streams/elections/) feature                        |
| `governor`      | Can use [Governors](https://choria.io/docs/streams/governor/)                                                              |
| `org_admin`     | Has access to all subjects in the organisation, full unlimited access on the broker                                        |
| `service`       | Is allowed to have a client JWT that's valid for a long time, typical for long running services that use the Choria Client |
| `system_user`   | Is allowed to access the Choria Broker system account for monitoring and management purposes  (> 0.26.0)                   |

### Server

During provisioning the server must have the setting `plugin.security.server_anon_tls=1`, it will also create `/etc/choria/server.seed` holding
the ED25519 private seed and `/etc/choria/server.jwt` holding the signed JWT file that will be presented to the broker.

### Client

The clent will not need TLS certificates, so it will set the `file` security provider:

```ini
plugin.security.provider = file
```

It will log into an AAA Service Login URL when running `choria login`:

```ini
plugin.choria.login.aasvc.login.url = https://aaa.example.net:4222/choria/v1/login
```

The `choria login` application will store a ED25519 private seed, sign requests using the AAA Signing Service and not
use mTLS to connect:

```ini
plugin.choria.security.request_signer.token_file = ~/.choria/token
plugin.choria.security.request_signer.service = true
plugin.choria.secutiry.client_anon_tls = true
```
