+++
title = "Next Steps"
weight = 150
toc = true
+++

At this point you should have a working Choria set up with your user able to do any command of the plugins we deployed. Follow the next steps to find other users or obtain support and validate your setup.

## Join the community

The community is where you can go to get support should you have any problems installing or using the software or just want to discuss ideas or concerns with other users.

We have a mailing list for users that will also receive notifications of updates to the modules and so forth. The group is hosted on [Google Groups, you can sign up here](https://groups.google.com/forum/#!forum/choria-users).

For more interactive comms I am on the Puppet IRC as _Volcane_ and on slack as _ripienaar_, we also have a _#choria_ channel on the [Puppet Slack](http://slack.puppet.com/) and [GitHub Discussions](https://github.com/choria-io/general/discussions).

If you wish to file a ticket about anything here or improve something you can do so in [choria-io/general](https://github.com/choria-io/general) where it will be triaged.

## Confirming it is working

You can inspect the resulting configuration on your CLI:

```nohighlight
$ mco choria show_config
Active Choria configuration:

The active configuration used in Choria comes from using Puppet AIO defaults, querying SRV
records and reading configuration files.  The below information shows the completely resolved
configuration that will be used when running MCollective commands

MCollective related:

    MCollective Version: 2.21.1
         Choria Version: 0.19.0
     Client Config File: /etc/puppetlabs/mcollective/client.cfg
     Active Config File: /etc/puppetlabs/mcollective/client.cfg
      Plugin Config Dir: /etc/puppetlabs/mcollective/plugin.d
      Using SRV Records: true
              Federated: false
             SRV Domain: example.com
               NATS NGS: false
     Middleware Servers: puppet.example.com:4222

Puppet related:

       Puppet Server: puppet.example.com:8140
     PuppetCA Server: puppet.example.com:8140
     PuppetDB Server: puppet.example.com:8081
     Discovery Proxy: not using a proxy
      Facter Command: /opt/puppetlabs/bin/facter
       Facter Domain: example.com

Security setup:

     Valid SSL Setup: yes
   Security Provider: puppet
            Certname: rip.mcollective
       SSL Directory: /home/rip/.puppetlabs/etc/puppet/ssl (found)
  Client Public Cert: /home/rip/.puppetlabs/etc/puppet/ssl/certs/rip.mcollective.pem (found)
  Client Private Key: /home/rip/.puppetlabs/etc/puppet/ssl/private_keys/rip.mcollective.pem (found)
             CA Path: /home/rip/.puppetlabs/etc/puppet/ssl/certs/ca.pem (found)
            CSR Path: /home/rip/.puppetlabs/etc/puppet/ssl/certificate_requests/rip.mcollective.pem (found)
      Public Cert CN: rip.mcollective (match)

Active Choria configuration settings as found in configuration files:

  choria.security.serializer: json
           choria.srv_domain: example.com

```

Here you'll see all results from your SRV records, config files, defaults etc all resolved.  You can run
this as root or a normal user and see it pick the right files etc

You can confirm you have some nodes connected to your collective - and get some information
about which middleware server they used in the case of clusters:

```nohighlight
$ mco rpc choria_util info

Discovering hosts using the mc method for 2 second(s) .... 47

 * [ ============================================================> ] 47 / 47



Summary of Choria Version:

   choria 0.18.0 = 42
   choria 0.14.0 = 3
   choria 0.17.0 = 2

Summary of Middleware Client Flavour:

   nats.go go1.14.10 = 42
    nats.go go1.14.2 = 3
    nats.go go1.14.9 = 2

Summary of Middleware Client Library Version:

   1.11.0 = 44
    1.9.2 = 3

Summary of Connected Broker:

   nats://puppet.example.com:4222 = 47

Summary of Connector TLS:

   true = 47

Summary of Protocol Secure:

   true = 47

Summary of SRV Domain:

   example.com = 47

Summary of SRV Used:

   true = 47


Finished processing 47 / 47 hosts in 666.77 ms

```

Here we observe 47 nodes, some nodes have older components than others but all are connected to *puppet.example.com:4222*.  Add *--display always* for much more details.

From the shell you set up the user in lets check the version of _puppet-agent_ installed on your nodes:

```nohighlight
$ mco package status puppet-agent

 * [ ============================================================> ] 3 / 3

                 dev1.example.net: puppet-agent-6.19.1-1.el7.x86_64
                 dev2.example.net: puppet-agent-6.19.1-1.el7.x86_64
                 dev3.example.net: puppet-agent-6.19.1-1.el7.x86_64

Summary of Arch:

   x86_64 = 3

Summary of Ensure:

   6.19.1-1.el7 = 3


Finished processing 3 / 3 hosts in 71.63 ms
```

Choria knows a lot about your nodes such as all their Facts and Puppet Classes deployed to them, you
can view what it knows about a certain node, this should include your classes, facts and agents deployed:

{{% notice tip %}}
Choria identities match the certificate names from Puppet
{{% /notice %}}

```nohighlight
$ mco inventory dev1.example.net

Inventory for dev1.example.net:

   Server Statistics:
                      Version: 0.18.0
                   Start Time: 2020-11-25 10:21:12 -1000
                  Config File: /etc/choria/server.conf
                  Collectives: mcollective
              Main Collective: mcollective
                   Process ID: 27374
               Total Messages: 339
      Messages Passed Filters: 291
            Messages Filtered: 48
             Expired Messages: 0
                 Replies Sent: 0
         Total Processor Time: 0 seconds
                  System Time: 0 seconds

   Agents:
      bolt_tasks      choria_util     discovery
      filemgr         nettest         package
      puppet          rpcutil         scout
      service         unix

   Data Plugins:
      No data plugins installed

   Configuration Management Classes:
    choria                               choria::broker
    choria::broker::config               choria::broker::service
    choria::config                       choria::install
    choria::repo                         choria::scout_checks
    choria::service

   Facts:
      aio_agent_version => 6.19.1
      architecture => x86_64
      augeas => {"version"=>"1.12.0"}
      augeasversion => 1.12.0
      ...
```

You can get a quick report of values for some fact (add -v for node names):

```nohighlight
$ mco facts aio_agent_version
Report for fact: aio_agent_version

        6.19.1                                   found 3 times

Finished processing 3 / 3 hosts in 18.61 ms
```

We can check that the Puppet service is up:

```nohighlight
$ mco service status puppet

 * [ ============================================================> ] 3 / 3

   dev3.example.net: running
   dev2.example.net: running
   dev1.example.net: running

Summary of Service Status:

   running = 3


Finished processing 3 / 3 hosts in 98.98 ms
```

And when last Puppet ran on these nodes:

```nohighlight
$ mco puppet status

 * [ ============================================================> ] 3 / 3

   dev1.example.net: Currently idling; last completed run 6 minutes 20 seconds ago
   dev2.example.net: Currently idling; last completed run 10 minutes 04 seconds ago
   dev3.example.net: Currently idling; last completed run 25 seconds ago

Summary of Applying:

   false = 3

Summary of Daemon Running:

   running = 3

Summary of Enabled:

   enabled = 3

Summary of Idling:

   true = 3

Summary of Status:

   idling = 3


Finished processing 3 / 3 hosts in 28.27 ms
```

## Further reading

There is much to learn about MCollective, at this point if all worked you'll have a setup
that is secured by using TLS on every middleware connection and every user securely identified
and authorized via the Puppet SSL certificates with auditing and logging.

Choria has a few optional features related to integration with PuppetDB and Puppet Server,
please see the [optional configuration](http://choria.io/docs/configuration/) section. You should
also read about [Choria Playbooks](http://choria.io/docs/playbooks).

The official documentation is a good resource for usage details:

  * Read about [Choria Playbooks](/docs/playbook) to write higher order orchestration scripts than the CLI provides
  * Read about [Puppet Tasks](/docs/tasks/) that provides a way to gain access to much more shared capabilities from the Puppet Forge
  * I strongly suggest you read about [using the mcollective CLI](../../concepts/cli)
  * Read about [filtering which hosts to act on](https://choria.io/docs/concepts/cli/#selecting-request-targets-using-filters)
  * Read about [managing Puppet](https://forge.puppet.com/choria/mcollective_agent_puppet) though note the setup steps are already completed
  * Read about [managing packages](https://forge.puppet.com/choria/mcollective_agent_package) though note the setup steps are already completed
  * Read about [managing services](https://forge.puppet.com/choria/mcollective_agent_service) though note the setup steps are already completed
  * Read about [managing files](https://forge.puppet.com/choria/mcollective_agent_filemgr) and this setup too is already completed

These plugins that manage Service, Package, Puppet and Files are just plugins for Choria.  You can write your own plugins,
deploy them and interact with them. There is an [official guide about this](https://choria.io/docs/development/).

If you do write your own plugins Choria helps you [package your plugins](../../configuration/plugins/) as Puppet modules like the ones we
just installed, sharable on the Forge.
