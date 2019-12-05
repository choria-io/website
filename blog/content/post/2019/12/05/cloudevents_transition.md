---
title: "Transitioning Events to CloudEvents"
date: 2019-12-05T9:09:00+01:00
tags: ["lifecycle", "events", "machines"]
draft: false
---

When we first introduced our [Lifecycle Events](https://choria.io/blog/post/2019/01/03/lifecycle/) we mentioned efforts by [cloudevents.io](https://cloudevents.io) to standardize a format and set of transports but that it was a bit early in their life to adopt it.

Cloud Events is now a CNCF Incubator project and they reached [CloudEvents 1.0](https://cloudevents.io/#v10) which means there is now a standard to work around that is stable. A good introduction to the concept can be found in [The CloudEvents Spec Seeks to Bring Uniformity to Event Data Descriptions](https://thenewstack.io/the-cloudevents-spec-seeks-to-bring-uniformity-to-event-data-descriptions/).

Since then adoption of this format has exploded - a whole bunch of companies and projects support ingesting and routing CloudEvents, examples: [solo.io](https://medium.com/solo-io/cloudevents-multi-cloud-and-the-gloo-between-them-5c1c7cfe4dce) with their [event powered API gateway](https://github.com/solo-io/gloo), [OpenFaaS](https://docs.openfaas.com/reference/triggers/), [Serverless.com Event Gateway](https://github.com/serverless/event-gateway), [Oracle Cloud](https://blogs.oracle.com/cloud-infrastructure/track-and-react-to-cloud-native-events), [Azure](https://azure.microsoft.com/en-us/blog/announcing-first-class-support-for-cloudevents-on-azure/), [Knative](https://knative.dev/development/serving/samples/cloudevents/cloudevents-go/) and many many more.

So for us the time is ripe to adjust the format of our events to support CloudEvents 1.0. As of the next release of the Choria Server the Life Cycle events and Autonomous Agent Events are all been transitioned to CloudEvents format.

In short a Cloud Event is a small bit of standard fluff that encapsulate a project specific event, here's an example startup event before CloudEvents:

```json
{
    "protocol":"io.choria.lifecycle.v1.shutdown",
    "id":"52af2efc-8a8d-4253-9b4d-859d0ae240fa",
    "identity":"example.net",
    "component":"server",
    "timestamp":1575536471
}
```

And here is the same event in CloudEvents format:

```json
{
    "id":"55f3a0b7-e467-4675-b392-0455b927fca6",
    "source":"io.choria.lifecycle",
    "specversion":"1.0",
    "subject":"example.net",
    "time":"2019-12-05T09:02:00Z",
    "type":"io.choria.lifecycle.v1.startup",
    "data": {
        "protocol":"io.choria.lifecycle.v1.startup",
        "id":"55f3a0b7-e467-4675-b392-0455b927fca6",
        "identity":"example.net",
        "component":"server",
        "timestamp":1575536520,
        "version":"0.12.1"
    }
}
```

You can see our whole format event is still there and Choria aware tools like the `choria tool event` CLI and others will automatically unpack these events and understand them and do the right thing, the data document will still validate against the same JSON Schemas as always. The wrapper fields being CloudEvents standard means these can be routed through many 3rd party systems using standard tooling.

Autonomous Agent events have had similar treatment, here's a state notification:

```json
{
    "id":"f637689b-bfbf-4456-88f7-d0896e097f2f",
    "source":"io.choria.machine",
    "specversion":"1.0",
    "subject":"example.net",
    "time":"2019-12-05T09:37:45Z",
    "type":"io.choria.machine.watcher.exec.v1.state",
    "data":{
        "protocol":"io.choria.machine.watcher.exec.v1.state",
        "identity":"example.net",
        "id":"bac83cb2-2a37-460b-9c7a-633c6286bff5",
        "version":"0.0.1",
        "timestamp":1575538665,
        "type":"exec",
        "machine":"DockerExample",
        "name":"monitor",
        "command":"./monitor.rb",
        "previous_outcome":"success",
        "previous_run_time":1387005527
    }
}
```

Today we only publish them to the Choria Broker NATS connection (CloudEvents has a NATS Transport), in time we'll consider adding support for other transports like webhooks.