+++
title = "Next Steps"
weight = 150
toc = true
+++

At this point you should have a working Choria set up with your user able to do any command of the plugins we deployed. Follow the next steps to find other users or obtain support and validate your setup.

## Join the community

The community is where you can go to get support should you have any problems installing or using the software or just want to discuss ideas or concerns with other users.

We have a mailing list for users that will also receive notifications of updates to the modules and so forth. The group is hosted on [Google Groups, you can sign up here](https://groups.google.com/forum/#!forum/choria-users).

For more interactive comms I am on the Puppet IRC as _Volcane_ and on slack as _ripienaar_, we also have a _#mcollective_ channel on Freenode of _#choria_ on the [Puppet Slack](http://slack.puppet.com/).

If you wish to file a ticket about anything here or improve something the 2 GitHub project are [choria-io/mcollective-choria](https://github.com/choria-io/mcollective-choria)
and [choria-io/puppet-choria](https://github.com/choria-io/puppet-choria) where you can file issues.

## Confirming it is working

You can inspect the resulting configuration on your CLI:

```bash
$ mco choria show_config
Active Choria configuration:

The active configuration used in Choria comes from using Puppet AIO defaults, querying SRV
records and reading configuration files.  The below information shows the completely resolved
configuration that will be used when running MCollective commands

MCollective selated:

 MCollective Version: 2.9.1
  Client Config File: /etc/puppetlabs/mcollective/client.cfg
  Active Config File: /etc/puppetlabs/mcollective/client.cfg
   Plugin Config Dir: /etc/puppetlabs/mcollective/plugin.d
   Using SRV Records: true
          SRV Domain: dev.example.net
  Middleware Servers: nats1.example.net:4222, nats2.example.net:4222, nats3.example.net:4222

Puppet related:

       Puppet Server: puppet1.example.net:8140
     PuppetCA Server: puppet1.example.net:8140
     PuppetDB Server: puppet1.example.net:8140
      Facter Command: /opt/puppetlabs/bin/facter
       Facter Domain: example.net

SSL setup:

     Valid SSL Setup: yes
            Certname: rip.mcollective
       SSL Directory: /home/rip/.puppetlabs/etc/puppet/ssl (found)
  Client Public Cert: /home/rip/.puppetlabs/etc/puppet/ssl/certs/rip.mcollective.pem (found)
  Client Private Key: /home/rip/.puppetlabs/etc/puppet/ssl/private_keys/rip.mcollective.pem (found)
             CA Path: /home/rip/.puppetlabs/etc/puppet/ssl/certs/ca.pem (found)
            CSR Path: /home/rip/.puppetlabs/etc/puppet/ssl/certificate_requests/rip.mcollective.pem (found)

Active Choria configuration settings as found in configuration files:

              choria.srv_domain: dev.example.net

```

Here you'll see all results from your SRV records, config files, defaults etc all resolved.  You can run
this as root or a normal user and see it pick the right files etc

You can confirm you have some nodes connected to your collective - and get some information
about which middleware server they used in the case of clusters:

```bash
$ mco rpc choria_util info

Discovering hosts using the choria method .... 3

 * [ ============================================================> ] 1 / 1



Summary of Client Flavour:

   nats-pure = 3

Summary of Client Version:

   0.2.0 = 3

Summary of Connected Broker:

   puppet1.example.net:4222 = 1
   puppet2.example.net:4222 = 2

Summary of SRV Domain:

   example.net = 2

Summary of SRV Used:

   true = 1


Finished processing 3 / 3 hosts in 228.40 ms
```

Here we observe 3 nodes with 1 connected to *puppet1.example.net:4222* and 2 connected to *puppet2.example.net:4222*.  Add *--display allways* for much more details.

From the shell you set up the user in lets check the version of _puppet-agent_ installed on your nodes:

```bash
$ mco package status puppet-agent

 * [ ============================================================> ] 15 / 15

                 dev1.example.net: puppet-agent-1.8.0-1.el7.x86_64
                 dev2.example.net: puppet-agent-1.8.0-1.el7.x86_64
                 dev3.example.net: puppet-agent-1.8.0-1.el7.x86_64

Summary of Arch:

   x86_64 = 3

Summary of Ensure:

   1.8.0-1.el7 = 3


Finished processing 3 / 3 hosts in 71.63 ms
```

MCollective knows a lot about your nodes such as all their Facts and Puppet Classes deployed to them, you
can view what it knows about a certain node, this should include your classes, facts and agents deployed:

{{% notice tip %}}
MCollective identities match the certificate names from Puppet
{{% /notice %}}

```bash
$ mco inventory dev1.example.net

Inventory for dev1.example.net:

   Server Statistics:
                      Version: 2.9.1
                   Start Time: 2016-12-16 21:35:20 +0100
                  Config File: /etc/puppetlabs/mcollective/server.cfg
                  Collectives: de_collective, mcollective
              Main Collective: mcollective
                   Process ID: 13127
               Total Messages: 12
      Messages Passed Filters: 12
            Messages Filtered: 0
             Expired Messages: 0
                 Replies Sent: 11
         Total Processor Time: 35.85 seconds
                  System Time: 16.33 seconds

   Agents:
      discovery       filemgr         package
      puppet          rpcutil         service

   Data Plugins:
      agent           collective      fact
      fstat           puppet          resource
      service

   Configuration Management Classes:
    mcollective                           mcollective::config
    mcollective::facts                    mcollective::packager
    mcollective::plugin_dirs              mcollective::service
    mcollective_agent_filemgr             mcollective_agent_iptables
    mcollective_agent_package             mcollective_agent_puppet
    mcollective_agent_puppetca            mcollective_agent_service
    mcollective_choria                    mcollective_util_actionpolicy

   Facts:
      aio_agent_version => 1.8.0
      architecture => x86_64
      augeas => {"version"=>"1.4.0"}
      augeasversion => 1.4.0
      ...
```

You can get a quick report of values for some fact (add -v for node names):

```bash
$ mco facts aio_agent_version
Report for fact: aio_agent_version

        1.8.0                                    found 3 times

Finished processing 3 / 3 hosts in 18.61 ms
```

We can check that the Puppet service is up:

```bash
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

```bash
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
  * Read about [filtering which hosts to act on](https://docs.puppet.com/mcollective/reference/ui/filters.html)
  * Read about [managing Puppet](https://github.com/puppetlabs/mcollective-puppet-agent#usage) though note the setup steps are already completed
  * Read about [managing packages](https://github.com/puppetlabs/mcollective-package-agent#readme) though note the setup steps are already completed
  * Read about [managing services](https://github.com/puppetlabs/mcollective-service-agent#readme) though note the setup steps are already completed
  * Read about [managing files](https://github.com/puppetlabs/mcollective-filemgr-agent#readme) and this setup too is already completed

These plugins that manage Service, Package, Puppet and Files are just plugins and they make the
main user facing feature set of Mcollective.  You can write your own plugins, deploy them and interact with
them.  There is an [official guide about this](https://docs.puppet.com/mcollective/simplerpc/agents.html).

If you do write your own plugins Choria helps you [package your plugins](../../configuration/plugins/) as Puppet modules like the ones we
just installed, sharable on the Forge.
