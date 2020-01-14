---
title: "Rego policies for Choria Server"
date: 2020-01-12T09:00:00+07:00
tags: ["security", "opa", "rego", "actionpolicy"]
draft: true
---

[Open Policy Agent](https://www.openpolicyagent.org/) is a CNCF incubating project that allow you to define Policy as code. It's widely used in various projects like Istio, Kubernetes and more.

It allows you to express authorization policies - like our [Action Policy](https://github.com/choria-plugins/action-policy) - in a much more flexible way.

Building on the work that was done for aaasvc, I've added a rego engine to the choria server, which will allow us to do most of what actionpolicy allows, as well as:

 * Assertions based on the arguments sent to actions
 * Assertions based on other request fields like TTL and Collective
 * Assertions based on if the server is set to provisioning mode or not

Read below the fold for our initial foray into OPA policies and what might come next.

<!--more-->
Lets look at an example OPA Policy as it relates to a specific user.

```rego
package io.choria.mcorpc.authpolicy

# it only checks `allow`, its good to default false
default allow = false

# User can deploy the "frontend" package, in collective "production"
allow {
	input.callerid = "choria=vjanelle.mcollective"
	input.action == "install"
	input.agent == "package"
	input.collective == "production"
	input.data.name == "frontend"
}

# can ask status anywhere in any environment
allow {
	input.action == "status"
	input.agent == "package"
}

# user can do anything the agent package in development
allow {
	input.agent == "package"
	input.collective == "development"
}
```

The context this will run in is the choria server - meaning each individual node.  Currently there is no bundle distribution mechanism(if you are familiar with OPA), so you will need to ensure that the rego policies are distributed to your nodes, much like the current actionpolicy.  OPA itself ships with a number of [built in functions](https://www.openpolicyagent.org/docs/latest/policy-reference/#built-in-functions) to further enhance the capabilities of your policies.  For example, you could limit actions to [certain times of the day](https://www.openpolicyagent.org/docs/latest/policy-reference/#time), or even issuing [http requests to other services](https://www.openpolicyagent.org/docs/latest/policy-reference/#http)

Within the policy, the following fields are present as input:

* `agent` - The [Agent struct](https://godoc.org/github.com/choria-io/mcorpc-agent-provider/mcorpc#Agent)
* `action` - A [MCORPC Request](https://godoc.org/github.com/choria-io/mcorpc-agent-provider/mcorpc#Request)
* `callerid` - The CallerID value from the Request
* `collective` - The Collective as defined in the Request
* `data` - Request Data, if any - all inputs sent to the action
* `ttl` - Time To Live, from the request
* `time` - The time the request was made
* `facts` - any facts present on the choria server
* `classes` - any classes present on the choria server
* `agents` - Additional agents present on the choria server
* `provision_mode` - If the choria server is compiled with provisioning mode enabled, and configured

This capability has been released with in [v0.13.1](https://github.com/choria-io/go-choria/releases/tag/v0.13.1), however it is still experimental.  There are [additional capabilities of Open Policy Agent](https://www.openpolicyagent.org/docs/latest/external-data/) which we have not enabled as of this writing.

If you're using the puppet modules to deploy choria and mcollective, you'll soon be able to specify the [`rego_policy_source`](https://github.com/choria-io/puppet-mcollective/blob/e82bd050ba4586089ac7091bb4b658a2bc4805b9/manifests/module_plugin.pp#L95) parameter to deploy a a managed policy file, on a per-agent basis.

To enable rego policy processing, you set the `rpcauthprovider` to `rego_policy`.  During development of policies, you can either change the `loglevel` to `debug`, or set the `plugin.regopolicy.tracing` configuration value to `true`.  This will provide enhanced information about policies and evaluations in the logs.