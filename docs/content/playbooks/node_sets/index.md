+++
title = "Node Sets"
weight = 420
toc = true
+++

Node Sets are how you tell the playbook about groups of nodes that exist on your network, today you can use MCollective Discovery, PQL Queries, YAML files, Shell Scripts and Terraform outputs to create the node sets.

In all cases the node sets must produce a list of *certnames* that will match what MCollective expect in cases where you want to use MCollective based tasks.  In future task types will be pluggable and you just have to produce whatever node sets your type of task needs.

## Common Node Set options

There are a few common options you can supply to all node sets, here's a sample *shell* discovery with some common options set, these apply to all discovery types:

```puppet
$nodes = choria::discover("shell",
  "script" => "/usr/local/bin/nodes.sh",
  "at_least" => 1,
  "when_empty" => "Could not find any nodes from nodes.sh",
  "limit" => 10,
  "test" => true,
  "uses" => {
    "appmgr" => "> 1.0.1"}
  }
)
```

|Option|Description|Sample|
|------|-----------|------|
|at_least|Fail if not at least this many nodes found|`at_least => 10`|
|when_empty|A custom error message when no nodes are found|`when_empty => "Could not find any web servers"`|
|limit|Accept only this many nodes from the discovery source, if more are found, discard the excess ones|`limit => 10`|
|test|When true performs a *mco rpc rpcutil ping* against the discovered nodes to ensure they are operational|`test => true`|
|uses|Use MCollective to audit the discovered nodes and ensure the stated agents conform to the SemVer specification given|

## MCollective Nodes

The most basic MCollective node set can be:

```puppet
$nodes = choria::discover(
  "discovery_method" => "choria",
  "test" => true,
  "classes" => ["apache"],
  "uses" => {"rpcutil" => "~ 1.0.0"},
  "facts" => ["countr=uk"]
)
```

You can also make it explicit that this is a *mcollective* type discovery:

```puppet
$nodes = choria::discover("mcollective",
  "discovery_method" => "choria",
  "test" => true,
  "classes" => ["apache"],
  "uses" => {"rpcutil" => "~ 1.0.0"}
)
```

This tells it to use the MCollective *choria* discovery method and since that is based on PuppetDB I set `test => true` to cause it to ping the nodes it discovered to verify their connectivity.

It uses a *classes* filter to limit the nodes, other examples below, and declare these nodes to use the *rpcutil* agent.

The *rpcutil* agent above in the *uses* section is specified as needing version *~ 1.0.0* - a SemVer range - and it will verify that this is true on the discovered nodes.

|Option|Description|Sample|
|------|-----------|------|
|discover_method|Which MCollective Discovery plugin to use, see *mco plugin doc* for a list|*discovery_method => choria*|
|identities|Discover nodes with these identities, corresponds to the MCollective CLI *-I* flag|*identities => ["/dev/"]*|
|classes|Discover nodes with these classes, corresponds to the MCollective CLI *-C* flag|*classes => ["apache"]*|
|agents|Discover nodes with these agents, corresponds to the MCollective CLI *-A* flag|*agents => ["puppet"]*|
|facts|Discover nodes with these facts, corresponds to the MCollective CLI *-F* flag|*facts => ["country=uk"]*|
|compound|Discover nodes that match a compound query, forces the *mc* discovery method, corresponds to the MCollective CLI *-S* flag|*compound => "fstat('/etc/hosts').md5=/baa3772104/ and environment=development"*|
|uses|A list of agents to this node set should have, references a previous declared agent in the *uses* section.  Will be confirmed prior to running the playbook|*uses => {"rpcutil" => "~ 1.0.0"}*|

## YAML Nodes

A YAML file is supported as a input format, it can have groups inside it like the example here:

```yaml
web_servers:
  - node1.example.net
  - node2.example.net

db_servers:
  - db1.example.net
  - db2.example.net
```

Put this in a file and you can use it as a Node Set source:

```puppet
$nodes = choria::discover("yaml",
  "group" => "web_servers",
  "source" => "/etc/your_co/nodes.yaml"
)
```

|Option|Description|Sample|
|------|-----------|------|
|group|A set of nodes declared in the YAML file|*group => web_servers*|
|source|Full path to a YAML file with your node sets|*source => /etc/your_co/nodes.yaml*|

## PQL Nodes

While the Choria discovery method supports PQL it's a bit strict on format, within the playbook system we have some more freedom so you can just make arbitrary PQL queries:

```puppet
$nodes = choria::discover("pql",
  "query" => "facts { name = 'country' and value = '${country}' }",
  "test" => true
)
```

All non deactivated certnames will be extracted and used as discovery source. This also shows how to use a input variable and a reminder you should test MCollectivity connectivity to this sort of node set

|Option|Description|Sample|
|------|-----------|------|
|query|A PQL query that should return *certname* as part of result sets|*query => "nodes { }"*|

### Shell Nodes

A shell script - or any program really - can be used to extract your nodes:

```puppet
$nodes = choria::discover("shell",
  "script" => "/usr/local/bin/nodes.sh"
)
```

The script should just output one certname per line. It supports any arguments you might need and of course you can use variable interpolation to put *inputs* there.

{{% notice warning %}}
I strongly suggest you validate any input you use as arguments here to not include things that might cause shell escapes.  You can do this using the `Choria::ShellSafe` data type.
{{% /notice %}}


|Option|Description|Sample|
|------|-----------|------|
|script|Shell command to run with any arguments etc|*script => "/usr/local/bin/nodes.sh"*|

### Terraform Nodes

Retrieves a Terraform output from a state file, only outputs of the *list* type are supported.  No effort to first pull remote states is currently made.

```puppet
$nodes = choria::discover("terraform",
  "statefile" => "/path/to/terraform.tfstate",
  "output" => "webservers"
)
```

This would run *terraform output -state /path/to/terraform.tfstate -json webservers*

|Option|Description|Sample|
|------|-----------|------|
|statefile|Path to a terraform statefile|*statefile => "/path/to/terraform.tfstate"*|
|terraform|Optional path to the terraform executable, path is checked otherwise|*terraform => /usr/local/bin/terraform*|
|output|The name of the defined output|*output => webservers*|
