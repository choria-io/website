+++
title = "Discovery"
weight = 204
toc = true
+++

Discovery in Choria is the system used to address nodes on the Choria network. For example when issuing the command *choria req service restart service=httpd -C apache* I am saying all machines tagged class with *apache* should have their *httpd* service restarted.

In the example above we discovered all nodes matching a specific criteria. This document will explore discovery in depth.

## Reliable or Best Efforts

Conceptually when managing large, dynamic, fleets of nodes or IoT devices it's very hard to maintain an up to date and richly decorated metadata service that could be queried about a near real time state of the network. This is because the network is constantly in flux. Users turning IoT devices on or off, administrators in other teams doing maintenance or hardware failing, disks filling up, there is always some number of transient and emergent behaviours in any sizable fleet.

A key concept of Choria is to be able to manage what is there now, without trying to access things that are not. Contrast this with a *webservers.txt* style file used to drive discovery, it's very hard to be accurate always and when trying to *ssh* to a disconnected host many seconds are wasted on machines that will never work.

The native Choria discovery system is tailored to this dynamic world, it can discover nodes and operate on the ones that's there now.  This promotes a style of administration where you build a backplane based on a continuous control loop of discover -> query -> remediate. A down machine is down, but if it ever comes back your loop will pick it up and remediate anything that requires it.  It's done efficiently and without wasting time on down machines.

In other scenarios where you do know and do care for a known set of machines - like when deploying a new version of your software that requires careful orchestration with database upgrades and more - it's vital to know what should be there and to know when a machine that should be there is not, or failed. Choria supports this by querying databases like PuppetDB or flat files or its own fleet inventory format.

In a sense this is like UDP and TCP, dynamic discovery does the best it can with what it has and lets you build resilient long term management backplanes that is stable in the face of uncertainty while the ability to query a data store for what should be there lets you build management systems with appropriate feedback when needed.

Choria supports a rich feature set in both these modes of discovery.

## Node Metadata

Choria maintains 2 sets of metadata about any machine. Nodes can be tagged using a list of items - we call them *classes* as borrowed from Puppet - this is simply a free-form list of words like *webserver*, *database*, *app_server* or Puppet classes like *profile::acme_server*. In the typical case this is supplied by Puppet but it is simply a text file with 1 word per line, you can source this from anywhere.

In addition to the *classes* we support facts, facts are YAML or JSON data for example *{"country":"uk"}*. We support arbitrarily nested data such as those produced by *facter*, you can supply any JSON or YAML structure.

Every single node maintains its own metadata store internally, requests directed at the fleet are evaluated against this metadata store before they are acted on.

Choria supports publishing the metadata it holds on a regular basis using a feature called *registration*, we have data adapters for *NATS Streaming Server* and *NATS JetStream* to make this data available to 3rd parties for analysis.

## Querying node metadata

{{% notice tip %}}
Most examples shown were made using our [Vagrant Demo](https://github.com/choria-io/vagrant-demo) environment
{{% /notice %}}

Given a fleet of Choria nodes we provide CLI tools and RPC APIs to query the metadata the network holds.

First let's look at one specific node using the *choria inventory <identity>* command:

```nohighlight
$ choria inventory choria0.choria
Inventory for choria0.choria

  Choria Server Statistics:

                    Version: 0.99.0.20210115
                 Start Time: 2021-01-15 14:19:17 +0000 UTC
                Config File: /etc/choria/server.conf
                Collectives: mcollective
            Main Collective: mcollective
...

  Configuration Management Classes:

    settings                          default
    roles::managed                    profiles::common
    mcollective                       mcollective::plugin_dirs
    mcollective::config               mcollective::facts
    mcollective_data_sysctl           mcollective_agent_shell
    mcollective_agent_process         mcollective_agent_nettest
    mcollective_agent_bolt_tasks      mcollective_choria
    mcollective_agent_puppet          mcollective_agent_service
    mcollective_agent_package         mcollective_agent_filemgr
    mcollective_util_actionpolicy     choria
    choria::repo                      choria::install
    choria::config                    choria::scout_checks
    choria::service                   choria::scout_metrics
    prometheus                        prometheus::node_exporter
    systemd                           systemd::systemctl::daemon_reload
    systemd::journald

  Facts:

    {
      "aio_agent_version": "6.19.1",
      "architecture": "x86_64",
      "os": {
        "architecture": "x86_64",
        "family": "RedHat",
        "hardware": "x86_64",
        "name": "CentOS",
        "release": {
          "full": "7.8.2003",
          "major": "7",
          "minor": "8"
        },
      },
    }
```

Above you see a truncated view of a node, it shows all tagged classes and all facts about this specific node. In this case data is from *facter* and *puppet*.

We can do a fleet wide report of a specific fact using the *choria facts <fact>* command, this is a real time view of the network, effectively treating the network as a data source:

```nohighlight
$ choria facts os.name --nodes
Report for fact: os.name

  CentOS found 3 times

    choria0.choria
    choria1.choria
    puppet.choria


Finished processing 3 / 3 hosts in 2.456s
```

Here *--nodes* ask it to show the matching node identities, *--table* will show tabular markdown format data and *--json* will output structured data.

Under the cover this is using the *rpcutil* agent *get_fact`* action, the raw data can be seen using *choria req rpcutil get_fact fact=os.name*.

Note the *os.name*, this is a [GJSON Path Syntax](https://github.com/tidwall/gjson/blob/master/SYNTAX.md) query across the nested facts, you can go deep into arrays and hashes using this format.

## Discovering nodes using basic attributes

Armed with knowledge of what is out there, and a way to see a fleet wide report of available values we can now look at how we can select nodes as targets for orchestration tasks.

In the section I will use *choria discover* to show matching nodes, these arguments are accepted on almost all *choria* and *mco* commands that operate on groups of nodes.

Basic tagged class discovery using *-C*, any nodes with exactly that class.

```nohighlight
$ choria discover -C choria
choria1.choria
choria0.choria
puppet.choria
```

Class discovery supports regular expressions, so *-C /choria|mcollective/* would work too.

Basic fact discovery using *-F*, any nodes with exactly that fact.

```nohighlight
$ choria discover -F os.name=centos
puppet.choria
choria0.choria
choria1.choria
```

Fact discovery supports a number of operators, *!=*, *<=*, *>=*, *>*, *<*, *=~* and is data type aware. Regular expression are supported as *os.name=~/entos/*, matching is not case-sensitive.

Boolean *and* discovery combining class and facts using *-W*.  This is a quick way to just combine multiple class and fact discoveries:

```nohighlight
$ choria discover -W "choria os.name=centos"
choria1.choria
choria0.choria
puppet.choria
```

This also supports regular expressions *-W "/^c/ os.name=~c.nt.s"*.

These queries are all supported by our native Choria network based discovery and PuppetDB based discovery, meaning you can use them in a reliable or best-efforts basis.

## PuppetDB PQL

PuppetDB has a rich query language called PQL, we support executing PQL queries as discovery as long as those queries return just lists of *certname*.

```nohighlight
$ choria discover -I 'pql:nodes[certname] { certname ~ ".choria" }' --dm=puppetdb
```

## Compound Filters

Things get more interesting when we look at something called Compound Filters. This is a new feature in the latest Choria Server. Previously MCollective had Compound Filters, but we've had to change the language to one that's more extendable and will grow with us.

We use a library called *expr* with its own [Language Definition](https://github.com/antonmedv/expr/blob/master/docs/Language-Definition.md), we augment this with GJSON based lookup for nested data to create something that can really go deep into your infrastructure.

These do not support querying PuppetDB - they only work with the network based discovery and the inventory file based discovery.

First a basic case - we want all machines for a certain customers staging environment or all machines with *prometheus* installed.

```nohighlight
$ choria discovery -S '(with("customer=acme") && with("environment=staging")) || with("/prometheus/")'
```

Within the expressions we have defined some variables:

| Variable  | Description                             |
|-----------|-----------------------------------------|
| `agents`  | List of known agents                    |
| `classes` | List of classes this machine belongs to |
| `facts`   | Facts for the machine as raw JSON       |

And we made a few functions available:

| Function  | Description                                                                                                                  |
|-----------|------------------------------------------------------------------------------------------------------------------------------|
| `with`    | Equivalent of a `-W` filter - class and fact matches combined with regular expression support                                |
| `fact`    | Retrieves a fact from the nested fact data using [GJSON path syntax](https://github.com/tidwall/gjson/blob/master/SYNTAX.md) |
| `include` | Checks if an array includes a specific element                                                                               |
| `semver`  | Performs semver version matching (since 0.26.0)                                                                              |

We can go really deep as here:

```nohighlight
with('apache') and                              # class or agent 'apache'
  with('/t.sting/') and                         # class or agent regex match 't.sting'
  with('fnumber=1.2') and                       # fact fnumber with a float value equals 1.2
  fact('nested.string') matches('h.llo') and    # lookup a fact 'nested.string' and regex match it with 'h.llo'
  include(fact('sarray'), '1') and              # check if the 'sarray' fact - a array of strings - include a value '1'
  include(fact('iarray'), 1) and                # check if the 'iarray' fact - a array of ints - include a value 1
  semver(fact('facterversion'), '>= 4.0.0')     # matches the version of facter to 
```

Here the *include* command is a basic function to check if an array contains something and *fact()* just looks up a fact using GJSON.

Lets dig deep into some data. Here's a typical facter networking fact (truncated):

```yaml
networking:
  interfaces:
  eth0:
    bindings:
      - address: 10.0.2.15
        netmask: 255.255.255.0
        network: 10.0.2.0
    bindings6:
      - address: fe80::5054:ff:fe4d:77d3
        netmask: 'ffff:ffff:ffff:ffff::'
        network: 'fe80::'
    dhcp: 10.0.2.2
    ip: 10.0.2.15
    ip6: fe80::5054:ff:fe4d:77d3
    mac: 52:54:00:4d:77:d3
    mtu: 1500
    netmask: 255.255.255.0
    netmask6: 'ffff:ffff:ffff:ffff::'
    network: 10.0.2.0
    network6: 'fe80::'
    scope6: link
  eth1:
    bindings:
      - address: 192.168.190.5
        netmask: 255.255.255.0
        network: 192.168.190.0
    bindings6:
      - address: fe80::a00:27ff:fe2b:7d42
        netmask: 'ffff:ffff:ffff:ffff::'
        network: 'fe80::'
    ip: 192.168.190.5
    ip6: fe80::a00:27ff:fe2b:7d42
    mac: '08:00:27:2b:7d:42'
    mtu: 1500
    netmask: 255.255.255.0
    netmask6: 'ffff:ffff:ffff:ffff::'
    network: 192.168.190.0
    network6: 'fe80::'
    scope6: link
```

The problem is machines can all have different NIC names - even multiple NICs and bonds - and any NIC can have many IP addresses bound. We would though love to discover all machines that on any binding belongs to a specific network:

```nohighlight
$ choria discover -S 'include(fact("networking.interfaces.*.bindings.#.network"), "10.0.2.0")'
choria1.choria
choria0.choria
puppet.choria
```

We get the all the networks across all NICs and all Bindings using GJSON, and then we check if any of them equals the *10.0.2.0* address.

It can be a bit tricky to get this right, we'll add some tooling to help try out a few queries in a future release.

## Data Providers

Data Providers expose real time state from the running node to the discovery system, today we support *choria*, *config_item* and *scout* Data Provider, in time users will be able to provide their own.

A Data Provider can be queried in a Compound filter when using the *broadcast* or *mc* discovery methods, here we find all machines where a specific Scout check is not *OK*:

```nohighlight
$ choria req service restart service=foo -S 'scout("check_foo").state != "OK"'
```

We can also discover all nodes connected to a specific broker and restart them (if provisioning is enabled):

```nohighlight
$ choria req choria_provision restart token=s3cret splay=10 -S 'choria().connected_broker match "choria1.example.net"'
```

The data that a Data Provider provides can be seen using the CLI:

```nohighlight
choria req rpcutil get_data source=choria -I choria1.choria
Discovering nodes using the mc method .... 1

1 / 1    0s [====================================================================] 100%

dev1.devco.net

        agents_count: 13
       classes_count: 95
         config_file: /etc/choria/server.conf
    connected_broker: nats://puppet.choria:4222
    expired_messages: 0
   filtered_messages: 1
    invalid_messages: 0
      machines_count: 17
     passed_messages: 15
        provisioning: false
      reply_messages: 14
      total_messages: 16
              uptime: 6010
      valid_messages: 15


Finished processing 1 / 1 hosts in 780ms
```

In time users will be able to add their own Data Providers, for the moment these are all that is available.

Some plugins take a query, like the *scout* one, here we find all nodes that had a CRITICAL state at any time in their last 10 checks:

```nohighlight
$ choria discover -S '"CRITICAL" in scout("check_puppet_run").history'
```

## Using RPC queries for discovery

Also in the latest Choria release we support the ability for the *choria req* command to do some Powershell inspired chaining of queries. This is also a feature MCollective had, one that required the *jgrep* utility to be installed, in Choria we will use our new *expr* based infrastructure to avoid this extra dependency.

Let's say we have a scenario where a specific version of PHP causes a problem, and we need to restart Apache nightly to deal with that leak while a better solution is found.  We don't want to restart the entire fleet and while we could query the fleet it would be a bunch of awkward *jq* to get this all working into a flat file of affected nodes.

We can though do a rpc query, filter its results using *expr* and then use the filtered result set as discovery source for a follow up rpc call.

```nohighlight
$ choria req package status package=php --json --filter-replies 'ok() && data("ensure")=="5.3.3-49.el6"' | \
     choria req service restart service=httpd
```

Here we use *expr* and the *--filter-replies* option to select out of all the replies received where the response indicated that the package status was successfully obtained and where the returned data key *ensure* matches a specific version.

We then pipe that result set as json into the *choria req service* to restart *httpd* on the matching machines.

Within the expression we have a few variables:

|Variable|Description|
|--------|-----------|
|`msg`   |The Statusmsg of the RPC reply|
|`code`  |The Statuscode of the RPC reply as an integer|

Within the expression we have a few functions, more might be added later:

|Function|Description|
|--------|-----------|
|`ok()`  |If the status code is `mcorpc.OK` (0)|
|`aborted()`|If the status code is `mcorpc.Aborted` (1)|
|`unknown_action()`|If the status code is `mcorpc.UnknownAction` (2)|
|`missing_data()`|If the status code is `mcorpc.MissingData` (3)|
|`invalid_data()`|If the status code is `mcorpc.InvalidData` (4)|
|`unknown_error()`|If the status code is `mcorpc.UnknownError` (5)|
|`data(query)`|Queries the reply data using GJSON Path Syntax|
|`include(hay, needle)`|Looks for needle in an array|
|`sender()`|The sender id where the reply is from|
|`time()`|The timestamp of the reply|

Today only the *choria req* command supports this behaviour, once we have it just right we'll extend it to all other choria commands.

## Discovery Methods

In Choria the various backends that implement discovery are called *Discovery Methods*, this section provides detail of each of the core ones. Most CLI tools support the `--dm` or `--discovery-method` option to pick a backend.

The following configuration options impact discovery backend selection and settings.

|Configuration Flag|Valid Options|Description|
|------------------|-------------|-----------|
|`default_discovery_method`|`mc`, `broadcast`, `puppetdb`, `choria`, `external`|When not specified on the CLI, this will be used|
|`discovery_timeout`|Integer in seconds|How long discovery is allowed to run, meaning might differ between methods|
|`default_discovery_options`||Options to pass to discovery plugins unless `--do` is set on the CLI|

## *broadcast* or *mc*

This is the default method of discovery, and the only one that is supported without any external dependencies. The Choria client sends an empty message with just a filter attached, all nodes that match the filter responds. We gather the replies, and those are the discovered nodes.

The `discovery_timeout` is how long the client waits for responses from the fleet after publishing the message asking for responses.

Supported Filters: *Class*, *Agent*, *Identity*, *Facts*, *Compound* and *Combined*

## *puppetdb* or *choria*

This method makes a request to PuppetDB with a PQL query structured to find nodes matching the filter query. 

|Configuration Flag|Valid Options|Description|
|------------------|-------------|-----------|
|`plugin.choria.puppetdb_host`|`puppet.example.net`|The hostname where PuppetDB can be found|
|`plugin.choria.puppetdb_port`|`8080`|The port the PuppetDB server listen on|
|`plugin.choria.srv_domain`|`example.net`|When SRV lookups are enable, the domain to find PuppetDB in|
|`plugin.choria.use_srv`|`true`|Enable or Disable SRV lookups|

When SRV lookups is enabled PuppetDB is resolved using a `_x-puppet-db._tcp.example.net` query.

## *flatfile*

The `flatfile` discovery method supports a number of file related discovery sources, you enable it by passing `--nodes`
to commands that supports that.

If an `-I` filter is supplied on the CLI only nodes matching that filter will be picked out of the file, regular expressions are supported.

It accepts the following run-time options, for example `choria req rpcutil ping --do 'filter=groups.#(name=="linux").targets'`

|Option|Description|
|------|-----------|
|`filter`|[GJSON path syntax](https://github.com/tidwall/gjson/blob/master/SYNTAX.md) query to dig into complex JSON or YAML|
|`file`|Sets the file to use|
|`format`|Sets the format - `json`, `yaml`, `yml`, `rpc`, `txt`|

### Text files

The most basic format is just a file with names like this:

```nohighlight
web1.example.net
web2.example.net
```

A command like `choria req rpcutil ping --nodes list.txt` will parse a file like above and discover those nodes. 

### JSON and YAML Files

When the `--nodes` argument ends in `json`, `yaml` or `yml` this format is automatically chosen, below are valid files.

```yaml
---
- web1.example.net
- web2.example.net
```

```json
[
  "web1.example.net",
  "web2.example.net"
]
```

These files are not particularly useful, but lets look at a more complex example:

```yaml
groups:
  - name: linux
    targets:
      - target1.example.com
      - target2.example.com
    config:
      transport: ssh
  - name: windows
    targets:
      - target3.example.com
      - target4.example.com
    config:
      transport: winrm
```

We can discover all the `targets` in the `linux` group using this command:

```nohighlight
$ choria discover --nodes bolt.yaml --do 'filter=groups.#(name=="linux").targets'
target1.example.com
target2.example.com
```

The filter here ues [GJSON path syntax](https://github.com/tidwall/gjson/blob/master/SYNTAX.md)

### *inventory*

Choria supports inventory files that holds within them full facts, agent lists, classes lists and collective membership information, enough to build rich discovery supporting our full feature set including Compound filters (without Data Providers).

Additionally, uniquely, Inventory files can hold named searched allowing you to save an often used set of discovery filters by name and reuse it.

Inventory files can be very large, an inventory of 10 000 nodes can take 180MB on disk as JSON data. While we think this feature is good for inventories with a few thousand nodes, a 10 000 node inventory works and can perform full compound search across all nodes in a few seconds.

#### Usage

General usage of Choria remains the same, all types of filter work including `-S` or Compound filters. There's one additional behaviour and that is if you filter for `-I group:named_group` the plugin will execute the stored discovery query against it's data.

#### Configuration

|Configuration Flag|Valid Options|Description|
|------------------|-------------|-----------|
|`plugin.choria.discovery.inventory.source`|`~/choria-inventory`|Path to the Inventory file|

It accepts the following run-time options, for example `choria req rpcutil ping --do 'filter=groups.#(name=="linux").targets'`

|Option|Description|
|------|-----------|
|`file`|The file to load the inventory from, overriding the configured setting|
|`noverify`|When set to any value will disable JSON Schema based verification|


#### Inventory Format

The inventory format is a YAML or JSON file that is strictly validated before use. File names must end in `.yaml`, `.yml` or `.json`.

{{% notice tip %}}
We publish a [JSON Schema of the inventory](https://choria.io/schemas/choria/discovery/v1/inventory_file.json), you can configure your editor to validate and assist your editing of this file
{{% /notice %}}

```yaml
$schema: https://choria.io/schemas/choria/discovery/v1/inventory_file.json

groups:
  - name: acme
    filter:
      facts:
        customer=acme
  - name: all
    filter:
      identities:
        - /./

nodes:
  - name: a1.example.net
    collectives:
      - acme
      - mcollective
    facts:
      customer=acme
    classes:
      apache
  - name: w1.example.net
    collectives:
      - widgets
      - mcollective
    facts:
      customer=widgets
    classes:
      mysql
```

This file will allow for commands like, essentially all discovery features will work:

 * all nodes in the `widgets` collective - `choria discover -T widgets`
 * all nodes for customer widgets - `choria discover -F customer=widgets`
 * all nodes for customer `acme` by using the node group `acme` - `choria discover -I group:acme`

When maintaining this command, after editing I suggest running `choria tool inventory --validate inventory.yaml` to make sure it's well formed.

#### Creating the inventory

Creating this file can be a pain, especially keeping it up to date. We have some initial tooling to help you with this, with more to come.

You can ask Choria to create or update this file for you, for example, if we just want to build the file based on an initial query we can do:

```
$ choria tool inventory inventory.yaml -W customer=acme
Discovering nodes .... 19

19 / 19    0s [====================================================================] 100%

Wrote 19 nodes to /home/rip/inventory.yaml
```

This will write a full inventory for all the discovered nodes and add an `all` group. Once you've made some changes to the file, like add more groups for example, Choria can update the file for you:

```
$ choria tool inventory inventory.yaml --update
```

This will do a fresh network query for node metadata, update the file and leave your groups alone.

Node this tool is doing discovery and then a `rpcutil#inventory` RPC request. Being that this is discovery driven you can discover nodes from any supported source, for example:

```
$ cat nodes.txt
a1.example.net
a2.example.net
$ choria tool inventory inventory.yaml --update --nodes nodes.txt
```

And, for full inception, you can use the current file as discovery - meaning it will try to update all the nodes in the file with new data:

```
$ choria tool inventory inventory.yaml --update --dm inventory --do file=inventory.yaml
```

### Choria Results 

Finally, this discovery method is also the one implementing the [RPC Request Chaining](#using-rpc-queries-for-discovery) for data on the CLI 

## *external*

The external method allow you to implement a Discovery Method using any programming language, and the Choria Client will execute your plugin when needed.

|Configuration Flag|Valid Options|Description|
|------------------|-------------|-----------|
|`plugin.choria.discovery.external.command`|`/some/command`|The command to run for discovery|

It accepts the following run-time options, for example `choria req rpcutil ping --do command=/another/command`

|Option|Description|
|------|-----------|
|`command`|Path to an alternative command overriding what is configured|

Additionally, any other options, excluding `command`, that gets passed this way will be sent to the external plugin in the request.

When run the command will be executed as `command <request file> <reply file> io.choria.choria.discovery.v1.external_request`, the following environment variables will be set:

|Variable|Description|
|--------|-----------|
|`CHORIA_EXTERNAL_REQUEST`|Where the request in JSON format can be found|
|`CHORIA_EXTERNAL_REPLY`|Where to write the response|
|`CHORIA_EXTERNAL_PROTOCOL`|`io.choria.choria.discovery.v1.external_request`|

The request will look like this:

```json
{
  "$schema": "https://choria.io/schemas/choria/discovery/v1/external_request.json",
  "protocol": "io.choria.choria.discovery.v1.external_request",
  "filter": {
    "fact": [{"fact": "country", "operator": "==","value": "mt"}],
    "cf_class": [],
    "agent": ["rpcutil"],
    "compound": [],
    "identity": []
  },
  "options": {},
  "collective": "mcollective",
  "timeout": 2
}
```

And the response can be either:

```json
{
  "protocol": "io.choria.choria.discovery.v1.external_reply",
  "nodes": ["n1.example.net"]
}
```

If there is a failure you can return:

```json
{
  "error": "Error shown to user"
}
```

For Golang the [External](https://github.com/choria-io/go-external), for Python use the [py-mco-agent](https://github.com/optiz0r/py-mco-agent) package can be used to easily implement a discovery source.
