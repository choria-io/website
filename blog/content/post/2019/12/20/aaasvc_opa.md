---
title: "Open Policy Agent in AAA Service"
date: 2019-12-20T09:00:00+01:00
tags: ["security", "opa"]
draft: false
---

[Open Policy Agent](https://www.openpolicyagent.org/) is a CNCF incubating project that allow you to define Policy as code. It's widely used in various projects like Istio, Kubernetes and more.

It allows you to express authorization policies - like our [Action Policy](https://github.com/choria-plugins/action-policy) - in a much more flexible way.

I've wanted to explore what is next for our general Authorization piece and I think this gets us to a very good place - and OPA have massive adoption so it's always good to adopt widely used standards.

Using Open Policy we'll be able to do a number of things we've never been able to do - but get asked about regularly:

 * Make sure requests have filters associated to avoid huge blast radius
 * Assertions based on the arguments sent to actions
 * Assertions based on other request fields like TTL and Collective

And just generally be much more expressive about it.

Read below the fold for our initial foray into OPA policies and what might come next.

<!--more-->
Lets look at an example OPA Policy as it relates to a specific user.

```rego
package choria.aaa.policy

# it only checks `allow`, its good to default false
default allow = false

# user can deploy only frontend of myco into production but only in malta
allow {
	input.action == "deploy"
	input.agent == "myco"
	input.data.component == "frontend"
	requires_fact_filter("country=mt")
	input.collective == "production"
}

# can ask status anywhere in any environment
allow {
	input.action == "status"
	input.agent == "myco"
}

# user can do anything myco related in development
allow {
	input.agent == "myco"
	input.collective == "development"
}
```

The context this will run in is our centralized AAA system [Choria Centralized AAA](https://github.com/choria-io/aaasvc). This system gets an unsigned request and signs it on behalf of the user. Effectively providing centralization of authentication, authorization and auditing - something that's a bit weird for Choria, but as you can see allow for some very expressive policies.

Here for example I ensure a user can do deploys in production but only in one very specific scenario - only in one country and only one app component.  To do this we inspect the filters assigned to the request, the input arguments to the action and things like the Sub Collective.  These are all parts that were never available to any authorizer so this is a great addition.

You can see we use `requires_fact_filter("country=mt")` here, this is a custom extension to the Rego language, we have a few:

 * `requires_filter()` - ensures that at least one of identity, class, compound of fact filters is not empty
 * `requires_fact_filter("country=mt")` - ensures the specific fact filter is present in the request
 * `requires_class_filter("apache")` - ensures the specific class filter is present in the request
 * `requires_identity_filter("some.node")` - ensures the specific identity filter is present in the request

And within the request you'll have access to these fields:

 * `agent` - the agent being invoked
 * `action` - the action being invoked
 * `data` - the contents of the request - all the inputs being sent to the action
 * `sender` - the sender host
 * `collective` - the targeted sub collective
 * `ttl` - the ttl of the request
 * `time` - the time the request was made
 * `site` - the site hosting the aaasvcs (from its config)
 * `claims` - all the JWT claims

This is a huge step forward especially as Rego allow for complex logic and even things like calling to other webservices etc.

In the case of AAA Service this policy gets embedded in a JWT token and signed by a central authority, any request the user makes anywhere in any environment will be subjected to this embedded policy which the user themselves cannot modify while being entirely stateless on the servers.  This is a huge step forward in my mind.

I've merged this but will play a bit more with it - next up we'll look at supporting node side policies using the same [Rego Policy Language](https://www.openpolicyagent.org/docs/latest/policy-language/), they'll have access to a bit less but still will represent a huge leap forward in our abilities.