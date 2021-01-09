+++
title = "First User"
weight = 140
toc = true
+++

Users who wish to manage nodes via Choria need to have certificates signed by the Puppet CA.  Choria includes a tool to request and manage these certificates.

## Create your first user

On the node you wish to run Choria CLI commands from you should have configured it as a _client_ in the previous step.

```bash
$ whoami
rip
```

When you are ready request the certificate from the CA, it will store it in the default location in _~/.puppetlabs_ as per Puppet AIO standards.

```bash
$ choria enroll
```

This will request a certificate from your Puppet CA, you should sign it there and once signed it will be downloaded and saved.  If you cannot sign it immediately you can safely run this command again later.

By default, as my username is _rip_ the certificate that was requested will be _rip.mcollective_.  The default Choria setup only allows _*.mcollective_ as certificate names.

You should now be able to run _mco ping_ and see some of your nodes:

```bash
$ choria ping
dev1.example.net                           time=55.35 ms
dev2.example.net                           time=57.67 ms
dev3.example.net                           time=59.52 ms
```

Most other commands will not work due to the _default deny_ nature of the Choria so you have to set up some Authorization rules.

## Authorization

As my user certificate is _rip.mcollective_ and I wish to be able to manage all aspects of my MCollective I am going to add a default allow rule to _Hiera_, add this to your common tier or whichever tier will select the nodes you wish to be able to manage:

{{% notice tip %}}
At the moment there is some redundancy and confusion between `mcollective` and `choria` modules, we will merge this into one soon but kept it this way to disrupt users as little as possible
{{% /notice %}}

```yaml
mcollective::site_policies:
  - action: "allow"
    callers: "choria=rip.mcollective"
    actions: "*"
    facts: "*"
    classes: "*"
```

Once this has been rolled out to your site you can go ahead and try commands like `mco puppet status`.

If you want to deploy further users I suggest you look at the [Choria AAA](../../configuration/aaa/) documentation section.
