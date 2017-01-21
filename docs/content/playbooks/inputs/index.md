+++
title = "Inputs"
weight = 410
+++

Inputs are how you get data from your CLI or data sources like Consul, etcd and so forth into your Playbooks.

Each input gets defined in a way quite similar to how options are written in MCollective and they will show up as CLI flags in *mco playbook*.

There are 2 types of inputs.  **static** inputs are provided on the CLI and does not change for the duration of the playbook run.  **dynamic** inputs are bound to a data source like Consul and read from there every time they are referenced.

A *dynamic* input can become *static* by specifically supplying it's data on the CLI.  You can also mark an input as being *dynamic_only* which means it can only be set by a data source and never the CLI.

Inputs are surfaced on the CLI scripts as shell arguments, you can give them a short description, expected type, validation and default values.

Input values can be referenced anywhere, except in inputs, via templates like *{{{ input.cluster }}}*.

Here are some examples:

This is a input called *cluster* that is a String and will be validated as such, it's required and has no default. The *:string* validation is the name of a MCollective validator plugin, so if you have custom ones you can use them:

```yaml
inputs:
  cluster:
    description: "Cluster to deploy"
    type: "String"
    required: true
    validation: ":string"
```

Here is a optional input with a default value, it also shows the use of the *shellsafe* validator that ensure this is safe to pass to scripts and would avoid shell injection attacks:

```yaml
inputs:
  cluster:
    description: "Cluster to deploy"
    type: "String"
    default: "alpha"
    validation: ":shellsafe"
```

Inputs can be sourced from data stores like *Consul*, *etcd* or local memory when they are not specifically given on the CLI, see the Data Sources page for full details:

```yaml
data_sources:
  local:
    type: memory

inputs:
  cluster:
    description: "Cluster to deploy"
    type: "String"
    default: "alpha"
    validation: ":shellsafe"
    data: "local/cluster"
```

The above sets up a local in memory data store and sets the input *cluster* to fetch data from there.  Should you not specify a value on the CLI it will consult the data store every time you reference the input.  When not found the *default* value is used.

The *cluster* input can be forced to be resolved only from the Data Source and never the CLI like this:

```yaml
inputs:
  cluster:
    description: "Cluster to deploy"
    data: "local/cluster"
    dynamic_only: true
```

[Data Sources](../data/) are an advanced topic and covered extensively in the dedicated [Data Sources](../data/) page.
