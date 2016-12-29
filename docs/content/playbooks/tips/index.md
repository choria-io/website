+++
title = "Tips"
weight = 450
+++

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

If you have a big *data* section you can make another macro and merge that in too, for example you could set the *nodes* in the macro and just supply *message*.  In this case with just 2 data items this is a waste though.

Use the *mco playbook show playbook.yaml* command to see if your merges are working.
