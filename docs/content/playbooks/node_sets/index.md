+++
title = "Node Sets"
weight = 420
+++

Node Sets are how you tell the playbook about groups of nodes that exist on your network, today you can use MCollective Discovery, PQL Queries, YAML files and Shell Scripts to create the node sets.

In all cases the node sets must produce a list of *certnames* that will match what MCollective expect in cases where you want to use MCollective based tasks.  In future task types will be pluggable and you just have to produce whatever node sets your type of task needs.

## Common Node Set options

There are a few common options you can supply to all node sets:

```yaml
nodes:
  servers:
    type: shell
    script: /usr/local/bin/nodes.sh
    at_least: 1
    when_empty: "Could not find any scripts from nodes.sh"
    limit: 10
```

|Option|Description|Sample|
|------|-----------|------|
|at_least|Fail if not at least this many nodes found|*at_least: 10*|
|when_empty|A custom error message when no nodes are found|*when_empty: "Could not find any web servers"|
|limit|Accept only this many nodes from the discovery source|*limit: 10*|

## MCollective Nodes

The most basic MCollective node set can be:

```yaml
uses:
  rpcutil: "~ 1.0.0"

nodes:
  servers:
    type: mcollective
    discovery_method: choria  # uses PuppetDB
    test: true                # does rpcutil ping on the found nodes
    classes:                  # basic class filter
      - apache
    uses:                     # validates the rpcutil agent exist as ~ 1.0.0
      - rpcutil
```

This tells it to use the MCollective *choria* discovery method and since that is based on PuppetDB I set *test: true* to cause it to ping the nodes it discovered to veriy their connectivity.

It uses a *classes* filter to limit the nodes, other examples below, and declare these nodes to use the *rpcutil* agent.

The *rpcutil* agent above in the *uses* section is specified as needing version *~ 1.0.0* - a SemVer range - and it will verify that this is true on the discovered nodes.

Discovery is same as on the CLI really, so fact filters would be:

```yaml
nodes:
  servers:
    type: mcollective
    facts:
      - "country=uk"
```

*agents*, *classes*, *facts* and  *identities* are supported as Arrays and the *compound* one is a single one you may use so its just a string.

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

```yaml
nodes:
  web_servers:
    type: yaml
    group: web_servers
    source: /etc/your_co/nodes.yaml

  db_servers:
    type: yaml
    group: db_servers
    source: /etc/your_co/nodes.yaml
```

## PQL Nodes

While the Choria discovery method supports PQL it's a bit strict on format, you can just make arbitrary PQL queries:

```yaml
nodes:
  uk_servers:
    type: pql
    query: "facts { name = 'country' and value = '{{{ input.country }}}' }"
    test: true
```

All non deactivated certnames will be extracted and used as discovery source. This also shows how to use a input variable and a reminder you should test MCollectivity connectivity to this sort of node set

### Shell Nodes

A shell script - or any program really - can be used to extract your nodes:

```yaml
nodes:
  servers:
    type: shell
    script: /usr/local/bin/nodes.sh
```

The script should just output one certname per line.  Today passing arguments to the script is not supported
