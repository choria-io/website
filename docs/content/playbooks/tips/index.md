+++
title = "Tips"
weight = 499
+++

## Nodes on the CLI

Generally nodes come from node sets but you can also use a input to mimic a node set to some extend.

```yaml
inputs:
  cluster:
    description: "Cluster to deploy"
    type: ":array"
    required: true

tasks:
  - mcollective:
      description: "Connectivity Test"
      action: "rpcutil.ping"
      nodes: "{{{ inputs.cluster }}}"
```

You can now supply nodes like this:

```
$ mco playbook run playbook.yaml --cluster web1.example.net --cluster web2.example.net
```

Here we have an *array* input that we use via the templates when specifying nodes, it's important in
this case that you replace the *test: true* behaviour that actual node sets have with a *rpc.ping* like
here.  Real node sets can also do things like validate the agents meet your expectation using *uses*
which this can not do.

So while I suggest you always use real node set, this can be a handy thing to do sometimes.

## Less repetition

At present the playbooks are YAML and YAML have various referencing and merging features.

You might have to do multiple webhook calls like in this example:

```yaml
hooks:
  on_success:
    - webhook:
        uri: https://hooks.example.net/webhook
        method: POST
        headers:
          "X-Acme-Token": "TOKEN"
        data:
          "message": "Deployed Acme release {{{ inputs.version }}}"
          "nodes": "{{{ nodes.servers }}}"

  on_fail:
    - webhook:
        uri: https://hooks.example.net/webhook
        method: POST
        headers:
          "X-Acme-Token": "TOKEN"
        data:
          "message": "Acme deployment to release {{{ inputs.version }}} failed"
          "nodes": "{{{ nodes.servers }}}"
```

This is quite annoying since all this stuff is mostly the same, you can create a kind of macro though which in larger playbooks can be really useful and time saving:

```yaml
macros:
  1: &ping_deploy_webhook
    uri: https://hooks.example.net/webhook
    method: POST
    headers:
      "X-Acme-Token": "TOKEN"
      "X-Acme-Request-ID": "{{{ uuid }}}"

hooks:
  on_success:
    - webhook:
        <<: *ping_deploy_webhook
        data:
          "message": "Deployed Acme release {{{ inputs.version }}}"
          "nodes": "{{{ nodes.servers }}}"

  on_fail:
    - webhook:
        <<: *ping_deploy_webhook
        data:
          "message": "Acme deployment to release {{{ inputs.version }}} failed"
          "nodes": "{{{ nodes.servers }}}"
```

{{% notice tip %}}
This *macros* key is not a formal standard thing but it's likely I will give it some special treatment - simply ignore it on purpose - so if you do wish to do this, stick to using *macros* like here.
{{% /notice %}}

If you have a big *data* section you can make another macro and merge that in too, for example you could set the *nodes* in the macro and just supply *message*.  In this case with just 2 data items this is a waste though.

Use the *mco playbook show playbook.yaml* command to see if your merges are working.

