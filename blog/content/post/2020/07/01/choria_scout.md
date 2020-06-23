---
title: "Introducing Choria Scout"
date: 2020-07-01T09:00:00+01:00
tags: ["releases"]
draft: true
---

## Overview

I recently had to refresh my infrastructure and revisited how many things are done - I now host all the Choria 
components on Kubernetes using our WIP [helm repos](https://github.com/choria-io/helm) and as a result also had to 
revisit monitoring for those components. Additionally I have a number of traditional hosts for nameservers, dev environment
and more.

For the traditional hosts I did not find any appealing solution that's free and realised I had almost 100% of what I need 
to build a pretty compelling monitoring platform and it will tie in nicely with the recent work I did to make Goss usable
as a Go Package. 

Today I'd like to introduce early work done in this regard, it's called *Choria Scout* and is a set of Choria Features 
to assist in building monitoring pipelines.

Current [choria.io](https://choria.io) is a distribution of Choria focused on Puppet users Scout will be a distribution
focused on monitoring. The Puppet user focused Choria will have the Scout features and a easy way to configure them but 
Scout will be able to self host and be tailored to the needs for monitoring agents. One should not need to use Choria
or enable the remote execution hosting features to be a Scout user.
 
## Features

As Choria is very framework focused, and really an operations integration platform at heart, I am not trying to solve
all the problems myself but focusing on maximising the integration possibilities.  So here's first a few screenshots 
of Dashboards and such and then I'll break down the features.

![](health-overview-small.png)

This dashboard shows overall views of checks, problems and have an additional expanded view showing selected checks
and issues found.

![](machine-watch.png)

This is a life view of the Cloud Events being emitted by the system.

```puppet
choria::scout_check{"check_typhon":
    plugin            => "/usr/lib64/nagios/plugins/check_procs",
    arguments         => "-C typhon -c 1:1",
    remediate_command => "service typhon restart",
}
```

Above creates a check which includes auto remediation.

Current main features:

 * Checks once configured runs without any central coordination and without requiring network connections
 * Supports the popular Nagios 4 exit code plugin API with Goss support planned
 * Support auto remediation of issues without any central coordination
 * Scalable to 100s of thousands of nodes
 * CLI observation tools to view live results stream and force or inhibit checks
 * API to set maintenance mode for any check or force immediate checks, subject to AAA, 2FA and OPA Policies with support 
   for authentication against Okta and other enterprise systems. Ruby and Golang supported.
 * Can integrate with any other Autonomous Agent features such as cron like 5-field schedules for when checks should run
 * Alerting through the Prometheus Alert Manager with integrations to Pager Duty etc
 * Check results can optionally be published to Prometheus Node Exporter to enable Prometheus + Grafana for over all 
   health, aggregate statuses and node level dashboards showing all checks and alerts for a node.
 * For non Prometheus integrations each state transition gets published as a CloudEvents v1.0 format message
 * Each check result (or just problem ones) are published as CloudEvents v1.0 format messages
 * Optional configuration via Puppet
 
Future features:

 * Additional related Autonomous Agent Watchers like integration into [Goss](https://github.com/aelsabbahy/goss)
 * Check Configuration updated without Puppet utilising NATS JetStream
 * Purpose build CLI manager
 * A purpose specific distribution of the features into a purpose built binary
   * Single binary install
   * Auto enrolling of new nodes into the infrastructure including Certificate Authorities managed by 
     Kubernetes Cert Manager
   * Suitable for deploying as side cars next to microservices in Kubernetes
   * Included event router to send Cloud Events to KNative, Azure Event Grid and similar systems

## Conclusion

This is the first in a series of posts about this feature. The core work to add Nagios check support to the Autonomous
Agents took about 4 hours and all the rest of the features are essentially things Choria already had and that just
happened by default.

Follow up posts will look at aspects of the system in detail and explore how this was built.
