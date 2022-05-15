+++
title = "Application Builder"
weight = 30
+++

Teams often find they have a number of `choria req` and other related commands that they execute frequently while 
managing a large Choria deployment.  These commands are hard to discover or share, frequently sat in runbooks where
they quickly get out of date.

We want to make it easy to create per team CLI macro style tools that invoke Choria RPC Agents, Interact with Key-Value
buckets or call other external helpers.

The Application Builder allows you to describe a CLI in a YAML file and will automatically build a `choria` like CLI
for you based on this input definition.

The application will feel comfortable within any Choria deployment and it's entire feature set will be discoverable 
using `--help`.

Initially we support calling Choria Agents, Interacting with Choria Key-Value Store and executing external commands.

{{% notice warning %}}
This is a proof of concept feature that will be introduced in Choria 0.26.0. It's subject to change and, we 
might even entirely remove it should this prove to be too restrictive as a replacement for the old application framework.
{{% /notice %}}


## Exploratory Example

Here we see a short example CLI built using this framework:

```nohighlight
$ opscmd
usage: opscmd [<flags>] <command> [<args> ...]

Example Inc Operations Tools 

Flags:
  --help  Show context-sensitive help (also try --help-long and
          --help-man).

Commands:
  help [<command>...]
    Show help.

  fleet info [<flags>]
    Get Choria Versions, broker distribution and more

  provisioning reprovision [<flags>]
    Re-provision running nodes

    Reprovisioning will remove the node from the network and reset it's
    configuration to factory default, it will then get a new
    configuration, credentials and more from the Choria Provisioner.

  provisioning restart [<flags>]
    Restarts running nodes

  provisioner force-election
    Force a leader election

  provisioner leader
    Obtain the name of the current cluster leader

  machines [<flags>]
    Query Autonomous Agents
```

We'll focus on the `machines` example a bit as it shows the current sweet spot behavior:

```nohighlight
$ opscmd machines --help
usage: opscmd machines [<flags>] <machine>

Query Autonomous Agents

Flags:
      --help                 Show context-sensitive help (also try
                             --help-long and --help-man).
      --senders              List only the names of matching nodes
      --json                 Render results as JSON
      --table                Render results as a table
      --display=DISPLAY      Display only a subset of results (ok, failed, all, none)
      --ne=VERSION           Matches instances that are not this version
      --le=VERSION           Matches instances that are older than or same as
      --lt=VERSION           Matches instances that are older than
      --ge=VERSION           Matches instances that are newer than or same as
      --gt=VERSION           Matches instances that are older than
      --eq=VERSION           Matches instances that are same as
      
Args:
  <machine>  Autonomous Agent to report on
```

We see we have a fairly standard `choria req` like CLI here, but, we have a number of extra flags like `--ne` and `--gt`.

Many of the standard fields here are opt-in, you can elect to not have discovery flags (not shown), filters, 
output formats (shown) or batching enabled (not shown).

When one runs: `opscmd machines example --ne 1.0.0 --senders` a list of nodes where the Autonomous Agent `example` is not
at version `1.0.0`.  This is equivalent to running:

```nohighlight
choria req choria_util machine_state machine=example --filter-replies 'ok() && semver(data("version"), "!= 1.0.0")' --senders
```

To create this command we had to create the following bit of YAML (abstract):

```yaml
name: opscmd
description: Example Inc Operations Tools
version: 0.0.1
author: https://wiki.example.com/

commands:
- name: machines
  description: Query Autonomous Agents
  type: rpc
  std_filters: false
  output_format_flags: true
  display_flag: true
  batch_flags: false
  request:
    agent: choria_util
    action: machine_state
    params:
      name: "{{.Arguments.machine}}"

  arguments:
    - name: machine
      description: Autonomous Agent to report on
      required: true

  flags:
    - name: ne
      description: Matches instances that are not this version
      place_holder: VERSION
      filter: ok() && semver(data("version"), "!= {{.Flags.ne}}")
```

We place this file in `/etc/choria/builder/opscmd-app.yaml` and create a symlink `/usr/bin/opscmd` that points to the
`choria` binary, anyone who invokes `opscmd` will now run the custom application.

## Configuration

A configuration file `applications.yaml` can be made that will be read and values made available in templates using
`{{ .Config.value | require "value is required in configuration" }}`.

The file should be a YAML file that is all string values.

```yaml
# example applications.yaml
password: s3cret
```

The configuration file and application definitions can be put in `.`, `~/.config/choria/builder` or `/etc/choria/builder`.

Once a definition is placed in any of the above locations in a file like `opscmd-app.yaml`, simply make a symlink from 
`/usr/bin/opscmd` (or anywhere in your path) to `/usr/bin/choria`.

## Templates

Some strings like commands, inputs and more are parsed via the Go template language. We have a few functions to help
with common tasks.

 * `require` - requires a string is set, mostly used for accessing configuration data `{{ .Config.value | require "value must be set"}}`, here we see how to set a custom error message
 * `escape` - perform shell escaping on the value, useful for executing external commands with user input `command: /bin/upgrade.sh "{{ .Arguments.version | escape }}"`
 * `read_filter` - reads the contents of a file, you can use this to read files given on the CLI as arguments or flags

## Reference

The Specification YAML file is validated using a JSON Schema, as a temporary location this Schema is in the 
[Choria source repository](https://raw.githubusercontent.com/choria-io/go-choria/main/internal/fs/schemas/builder.json), 
once this feature is complete it will be stored in our usual schema repository.

Each Specification must have these fields:

```yaml
# CLI tool name, must be all lower case, no spaces or numbers. Can have dash and underscore
name: opscmd

# Human friendly description shown on initial --help page
description: Example Inc Operations Tools

# Version reported in opscmd --version
version: 0.0.1

# Any author hint, opscmd --help-man makes a man page that includes this
author: https://wiki.example.com/
```

### Generic Command

All commands types have at least these behaviors so we'll show the generic options and then what makes commands types 
unique in specific sections.

```yaml
commands:
  - name: example
    description: An example sub command
    # Can also be called as 'opscmd ex' or 'opscmd e' 
    aliases:
      - ex
      - e
    # The type of command this is, one of rpc, exec, kv or parent
    type: x
    commands: [] # sub commands that are any valid command

    # Prompts for a y/n before performing the action, no prompt if not set
    confirm_prompt: Do you really want to perform this action
    
    # arguments are like 'opscmd command argument' they do not require dashes before and
    # tend to be reserved for things that are required, though the last ones can all be
    # optional
    arguments:
      - name: item
        description: The item to act on
        required: true

    # Flags are passed like --value=x, today we only support string value flags, they can
    # can be required if needed
    flags:
      - name: value
        description: Sets a value
        # in --help this looks like --value=VALUE by default, --value=VAL with this
        placeholder: VAL
        required: false
```

### `parent` Command Type

The `parent` command type does not do anything, it just allows you to create a set of related sub commands.

Arguments and Flags are ignored.

```yaml
commands:
  - name: provisioning
    description: Fleet provisioning actions
    type: parent
    commands: [] # here list rpc, kv, exec or parent commands. 
```

### `exec` Command Type

The `exec` command type will just run some other command. When the limitations placed on this framework prevents
you from doing something, or you just have a shell script already to do something, you can call them from here.

In the example below when someone runs `opscmd install --version 1.2.3` it will invoke `/usr/local/bin/install.rb 1.2.3`,
the version argument is required.

```yaml
commands:
  - name: install
    description: Installs and Configured something
    type: exec
    command: /usr/local/bin/install.rb --version {{ .Arguments.version }}
    arguments:
      - name: version
        description: The version to install
        required: true
```

### `kv` Command Type

The `kv` command can Put, Get, Delete or History data from a Choria Key-Value bucket. 

The example below will show a current leader for something using Choria Leader Election and force a Leader Election by
deleting some data.  It also shows creating some data.

When creating data we support template expansion to read values from `.Flags`, `.Arguments` or `.Configuration`.

```yaml
commands:
  - name: app
    description: Manages app
    type: parent
    commands:
      - name: leader
        description: Obtain the name of the current cluster leader
        type: kv
        action: get
        bucket: CHORIA_LEADER_ELECTION
        key: app

      - name: force-election
        description: Force a leader election
        type: kv
        action: del
        bucket: CHORIA_LEADER_ELECTION
        key: app

      - name: upgrade
        description: Triggers an upgrade to a new version
        type: kv
        action: put
        bucket: APP_CONFIG
        key: version
        value: "{{ .Arguments.version }}"
        arguments:
          - name: version
            description: The version to deploy
            required: true

      - name: history
        description: View recent deployed versions
        type: kv
        action: history
        bucket: APP_CONFIG
        key: version
```

### `rpc` Command Type

The `rpc` command type invokes Choria RPC Agents, it renders data like `choria req` and supports almost 100% compatible 
flags as `choria req`, essentially it's a frontend with configurable defaults.

In the example below we initial the `machine_state` action on `choria_util` agent with properties built from command line
arguments, flags, and we set some defaults.

 * `std_filters` will enable `-C`, `-I`, `-S`, `-F` and friends
 * `output_format_flags` will enable `--json`, `--table`, `--senders` etc
 * `display_flag` will enable `--display`
 * `batch_flags` will enable `--batch` and `--batch-sleep`
 * `all_nodes_confirm_prompt` will prompt for confirmation when no filters are given - meaning all nodes will be acted on
 
All of these match their `choria req` counter-parts and all are off by default.

This type will have a number of features added to it over the following days including the ability to force display, 
batching, output format and filters
 
```yaml
commands:
  - name: machines
    description: Query Autonomous Agents
    type: rpc
    std_filters: true
    output_format_flags: true
    display_flag: true
    batch_flags: true
    all_nodes_confirm_prompt: Really act on all nodes without a filter
    request:
      agent: choria_util
      action: machine_state
      inputs:
        name: "{{.Arguments.machine}}"

    arguments:
      - name: machine
        description: Autonomous Agent to report on
        required: true

    flags:
      - name: ne
        description: Matches instances that are not this version
        place_holder: VERSION
        filter: ok() && semver(data("version"), "!= {{.Flags.ne}}")
```

While above, we enabled flags like `--batch`, we can instead disable `--batch` entirely and force it to a specific value,
we can do the same with a few other options shown here.  In all cases except `filter` when you set it specifically you cannot
also set the option to enable matching flags from being set, in the case of `filter` it will merge with `std_filter`:

```yaml
commands:
  - name: machines
    description: Query Autonomous Agents
    type: rpc
    
    # sets batch settings, prevents batch_flags from being added to the cli options 
    batch: 10
    batch_sleep: 60

    # sets display mode overriding the DDL, prevents display_flag from being used
    display: none
    
    # disables the progress bar
    no_progress: true
    
    # forces JSON output, prevents output_format_flags from being used
    output_format: json
    
    # sets specific discovery filters and options, will be merged with std_options
    filter:
      facts: [operatingsystem=CentOS]
      discovery_method: choria
      classes: [puppet]

    # as before
```
