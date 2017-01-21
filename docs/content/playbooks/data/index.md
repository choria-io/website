+++
title = "Data Sources"
weight = 450
toc = true
+++

Data Sources allow you to dynamically read and write data from tools like Consul, etcd or a local memory space.

In future it will be possible to do playbook level locks and task level locks against these tools too, thus coording across multiple instances of the same playbook or related tasks across different playbooks.  You will also be able to extract service membership from for example Consul Services.

Today the only integration that exist is in Inputs and Tasks.

{{% notice tip %}}
This feature is included since *0.0.20*
{{% /notice %}}

### Defining a Data Source

You define a data source with a unique name and source specific properties, you can then reference it in inputs, tasks etc by name.

```yaml
data_sources:
  local_memory:
    type: memory
```

This creates a data source called *local_memory*, for the moment only a basic local to the playbook memory store exist.

{{% notice tip %}}
You can use templates when defining Data Sources so you can use inputs to vary hostnames, usernames etc
{{% /notice %}}

### Binding inputs to Data Sources

When you define an input and add the *data* key to it this should reference a previously defined data source.

```yaml
inputs:
  cluster:
    description: "Cluster to deploy"
    type: "String"
    default: "alpha"
    validation: ":shellsafe"
    data: "local_memory/cluster"
```

You'll still be able to supply the input on the CLI - in which case it becomes static and does not change for the life of the playbook - but if you do not it will bind to the key *cluster* in the data store called *local_memory*.  If you add the key *dynamic_only* and set it to true then the input will not appear on the CLI and will only resolve from data sources and defaults

From then on any time you reference it in a template like *{{{ inputs.cluster }}}* it will query the data store and will not cache this result.  So if the playbook or an extenal tool adjusts the data in the data source the playbook will always get current data.

For the moment only String data is supported by the data sources, but validation will happen like *:shellsafe* here.

### Editing Data Sources

A playbook can write and delete data using the new *data* task - see the [Tasks](../tasks/) reference for full details.

```yaml
tasks:
  - shell:
      command: "/usr/local/bin/deploy --cluster {{{ inputs.cluster }}}"

  - data:
      action: "write"
      key: "local_memory/cluster"
      value: "bravo"

  - shell:
      command: "/usr/local/bin/deploy --cluster {{{ inputs.cluster }}}"
```

Here when the *cluster* is dynamic - not given on the CLI - it will default first to *alpha* and then get adjusted by the playbook to *bravo*.

Put in a few extra steps between the two to validate your deploys against the *alpha* cluster and you can do a blue/green deploy in the same playbook.  The 2 shell tasks will have a different value for the *cluster* input.

### Reports

Your Reports will include both *dynamic* and *static* inputs.  In the case of dynamic ones you will only get a list of the inputs that were dynamic and not their data since the data cannot be known and have different values for the life of the playbook.
