+++
title = "Inputs"
weight = 410
+++

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

[Data Sources](../data/) are an advanced topic and covered extensively in the dedicated Data Sources page.
