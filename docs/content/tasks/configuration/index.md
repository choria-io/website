+++
title = "Configuration"
weight = 100
+++

There are a number of steps you have to perform, which includes changes to your Puppet Server, to get Tasks working.  For this reason it's not enabled by default, the following will guide you through the required steps.

## Puppet Server

Your Puppet Server have to be quite recent - at least version 5.0.0 but 5.1.x or newer is best.

You have to enable the Tasks file end points and ensure all your nodes are authenticated to read from them. The Choria client and Servers will fetch Task Metadata and Task Files from the Puppet Server.

I use the [puppet_authorization](https://forge.puppet.com/puppetlabs/puppet_authorization) module in the example, add rules as follows:

```puppet
puppet_authorization::rule { "puppetlabs tasks file contents":
  match_request_path   => "/puppet/v3/file_content/tasks",
  match_request_type   => "path",
  match_request_method => "get",
  allow                => ["*"],
  sort_order           => 510,
  path                 => "/etc/puppetlabs/puppetserver/conf.d/auth.conf",
}

puppet_authorization::rule { "puppetlabs tasks":
  match_request_path   => "/puppet/v3/tasks",
  match_request_type   => "path",
  match_request_method => "get",
  allow                => ["*"],
  sort_order           => 510,
  path                 => "/etc/puppetlabs/puppetserver/conf.d/auth.conf",
}
```

## Servers and Clients

You have to install an extra plugin in your environment which includes the Task helpers

```yaml
mcollective::plugin_classes:
  - mcollective_agent_bolt_tasks
```

## RBAC

Basic RBAC rules are shown here, but refer to a later section in this guide for further details and tips about RBAC for Tasks

```yaml
mcollective_agent_bolt_tasks::policies:
  - action: "allow"
    callers: "choria=rip.mcollective"
    actions: "*"
    facts: "*"
    classes: "*"
```

Change *choria=rip.mcollective* here with your own certificate name, this will give you full control of the tasks feature and all tasks.

## Obtain some tasks

Tasks are delivered using Puppet modules much like anything else in the Puppet world. Uniquely to Tasks you only have to put the files on your Puppet Server module paths, you do not need to include any classes etc.

{{% notice tip %}}
At present Choria will only consult your _production_ environment for tasks.
{{% /notice %}}

You can therefore use _puppet module_, _r10k_ or _librarian puppet_ to place your modules in the _production_ environment and that should be enough for them to be used by Choria

## End to End Testing

A test task is inluded in the _mcollective\_agent\_bolt\_tasks_ module, you can verify the functionality of your network using it:

<pre><code class="nohighlight">
$ mco tasks run mcollective_agent_bolt_tasks::ping --message "hello world"
Retrieving task metadata for task mcollective_agent_bolt_tasks::ping from the Puppet Server
Attempting to download and run task mcollective_agent_bolt_tasks::ping on 33 nodes

Downloading and verifying 1 file(s) from the Puppet Server to all nodes: ✓  33 / 33
Running task mcollective_agent_bolt_tasks::ping and waiting up to 60 seconds for it to complete



Summary for task 884525e46b015b0789e57c019cd5f990

                       Task Name: mcollective_agent_bolt_tasks::ping
                          Caller: choria=rip.mcollective
                       Completed: 33
                         Running: 0

                      Successful: 33
                          Failed: 0

                Average Run Time: 0.13s
</code></pre>

After execution you can retrieve the output of each command:

<pre><code class="nohighlight">
$ mco tasks status 884525e46b015b0789e57c019cd5f990 -v
Discovering hosts using the choria method .... 33

node1.example.net
   {"message":"hello world","timestamp":"2018-03-19 13:21:18 +0000"}

.......

Summary for task 884525e46b015b0789e57c019cd5f990

                       Task Name: mcollective_agent_bolt_tasks::ping
                          Caller: choria=rip.mcollective
                       Completed: 33
                         Running: 0

                      Successful: 33
                          Failed: 0

                Average Run Time: 0.13s
</code></pre>

You may also see the results in JSON format with the *-j* flag.

## Next Steps

At this point your tasks feature is working, you have a number of next steps to follow:

  * Review the [Usage](../usage/) documentation section
  * Create RBAC fules for your team and review [general security considerations](../security/)
  * [Write your own Tasks](https://puppet.com/docs/bolt/0.x/writing_tasks.html) or [Find some on the Forge](https://forge.puppet.com/modules?utf-8=%E2%9C%93&sort=rank&q=&endorsements=&with_tasks=yes)

