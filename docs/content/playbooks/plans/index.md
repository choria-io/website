+++
title = "Puppet Plans DSL"
weight = 470
+++

Puppet Plans are a brand new feature - introduced October 2017 - that lets you write workflows using the Puppet DSL.

They are designed to be used with the Bolt orchestrator - a SSH based task runner from Puppet Inc.

The system though is extendible and already a plethora of modules on the forge includes tasks and integrations.

We created a set of functions for Choria that lets you write Playbooks using the Plans DSL and it allows for greater interaction with node results and integration with a whole new slew of 3rd party systems via Forge modules.

Using plans have a number of advantages over the Playbook YAML system, however this is still effectively a Technology Preview both from Puppet (the plans feature) and from Choria (the integration):

  * You can write functions and reuse them between plans, things like disabling puppet and waiting for all to go idle can now be reused easily
  * You have better control over branching and error conditions as you can itterate over the results from Choria agents etc
  * You can ship compatible functions, plans etc in your Puppet modules
  * You do not need to learn a new crazy YAML thing
  * You can use your existing tooling to document and test plans
  * You can mix in many more integrations like those from Google for interacting with GCP for example

I would say for a typical interactive Playbook this is the best way to write them.  Down the line I expect the YAML format to be the underlying data format for a Playbook Service but that is some way off.

{{% notice warning %}}
Please note that both the Plans feature in Puppet and the Integration from Choria is experimental or Feature Preview.  For sure I expect the Choria feature to only ship once Bolt does not vendor it's own Puppet.
{{% /notice %}}

## Discovering Nodes

You can discover nodes using any supported [Node Set](https://choria.io/docs/playbooks/node_sets/) in Playbooks like via Terraform outputs:

```puppet
$acme_servers = choria_discover("terraform",
  statefile => "/path/to/terraform.tfstate",
  output => "acme_servers"
)
```

You can go further and stick discovery in a function so you can reuse it between plans as `acme_app::webservers()`, and ship it via a module.  Here using Choria Discovery via PuppetDB:

```puppet
function acme_app::webservers (
  Optional[Enum[alpha, bravo]] $cluster = undef
) {
  $dopt = {
    discovery_method => "choria",
    agent => ["acme"]
  }

  $facts = $cluster ? {
    String => {"facts" => ["cluster=${cluster}"]},
    default => {}
  }

  choria_discover(
    $dopt + $facts
  )
}
```

Validating Choria agent versions via the Uses clause is a bit different here but still available.  The below code will find all webserver nodes and using the Choria RPC protocol inventory the nodes and ensure they all run a new enough version of the Puppet agent that supports what you want to do:

```puppet
choria_discover(
  facts => ["role=webserver"],
  uses => {
    puppet => "> 1.0.0"
  }
)
```

## Tasks

Generally all the [Task Types](https://choria.io/docs/playbooks/tasks/) are available to use in a Plan. Some like the Bolt Task makes no sense since you'd rather just use `run_task()` and likewise the Data task can be avoided since you can interact with the data directly (see below).

When the Choria action fails the Plan will exit with an error:

```puppet
choria_task("mcollective",
  action => "puppet.enable",
  nodes => $nodes
)
```

You can pass the `fail_ok` option to instead of failing return a Execution Result so you can do your own error handling or data processing:

```puppet
$result = choria_task("webhook",
  fail_ok => true,
  uri => "http://integration.example.net/",
  method => "GET",
  headers => {
    "X-Plan-Run" => "1"
  },
  data => {
    message => "Deployed version ${version} via Puppet plan",
    nodes => $nodes.join(", ")
  }
)

if $result.ok {
  notice(sprintf("Submitted webhook: response: %s", $result.value("integration.example.net")))
} else {
  fail(sprintf("Could not submit webhook to integration.example.net: %s", $result.partial_result))
}
```

You can interact with Choria RPC results via the Execution Result API, this will give you much greater freedom to plan your own error handling than the task list based approach in the Playbook YAML document.

```puppet
$status = choria_task("mcollective",
  action => "puppet.status",
  nodes => $nodes
)

$status.ok_nodes.each |$node, $status| {
  notice(sprintf("%s: %s", $node, $status["data"]["message"]))
}
```

## Reading / Writing Data

Data for this kind of interaction is a bit different from Hiera data, since execution is against remote nodes you might find you want to expose data via a network addressible data store.  Consul, Vault and Etcd are good options for this.

You can write to a consul data store:

```puppet
$ds = {
  type => "consul",
  timeout => 120,
  ttl => 60
}

choria_data("version", $version, $ds)
```

The data will now be in Consul where any other network based system that might be integrated in Consul can find it, you can also read it:

```puppet
$version = choria_data("version", $ds)
```

Another handy feature of these systems is network wide locking.  Upgrading an applications data base is quite sensitive, you might be doing this via Plans but what if 2 operators are not aware of each other and run the update at the same time? Choria let you use a global lock in a Plan to manage this.  Here the lock is stored in Consul and anyone with access to the same Consul DC will be coordinated around this lock:

```puppet
$ds = {
  type => "consul",
  timeout => 120,
  ttl => 60
}

choria_lock("db_update", $ds) || {
  choria_task(
    fail_ok => true,
    action => "puppet.disable",
    nodes => $servers
  )

  run_task("acme::db_upgrade", $servers)

  choria_task(
    action => "puppet.enable",
    nodes => $servers
  )
}
```

Or you perhaps just want to maintain a `~/.plans.rc` with some commonly used pieces of information:

```puppet
function bolt_rc(
  String $item,
  Optional[String] $value
) {
  $ds = {
    type => "file",
    file => "~/.plans.rc",
    format => "yaml',
    create => true
  }

  if $value {
    choria_data($item, $value, $ds)
  } else {
    choria_data($item, $ds)
  }
}

#....

# reads the jira API from your rc
$slack_token = bolt_rc("slack_token")

# saves some other value
bolt_rc("last_version", $version)
```

## Setup

To get going you have to install Bolt into your Puppet install that already has a working MCollective.  At present Bolt vendors it's own Puppet, Facter and Hiera so I'd consider this whole thing a bit risky.  Pick a machine that you do not care too much for, like don't use your Puppet Master.

On my machines Bolt just would not work because Facter 2 is buggy and it's ec2 fact would just sit and try to connect to thing that will never work, so your milage might vary.  Puppet will hopefully soon stop with this vendoring.

```shell
sudo /opt/puppetlabs/puppet/bin/gem install bolt
```

Now create a directory for your bolt setup and set up some modules you will need there:

```shell
mkdir work/bolts
cd work/bolts
puppet module install choria/mcollective_choria --target .
```

Now you'll need a module to hold your plans, obviously call this what works for you but to play around:

```shell
mkdir work/bolts/mymod/plans -p
cd work/bolts
```

Edit the file `mymod/plans/init.pp` and place in it:

```puppet
plan mymod {
  $result = choria_task(
    action => "puppet.status",
    nodes  => choria_discover("agent" => ["puppet"])
  )

  $result.ok_nodes.each |$node, $status| {
    notice(sprintf("%s: %s", $node, $status["data"]["message"]))
  }
}
```

Verify you have some nodes that has the Puppet agent - all should have if you have a working Choria machine:

```shell
mco puppet status
```

{{% notice tip %}}
If this command appears to hand it might be the broken ec2 facts, rm `/opt/puppetlabs/puppet/lib/ruby/gems/2.4.0/gems/bolt-0.5.1/vendored/facter/lib/facter/ec2.rb`
{{% /notice %}}

And run the plan like this, the RUBYLIB thing should go away once Puppet stop vendoring Puppet into Bolt:

```shell
export RUBYLIB=/opt/puppetlabs/mcollective/plugins
/opt/puppetlabs/puppet/bin/bolt plan run mymod --modules .
```

