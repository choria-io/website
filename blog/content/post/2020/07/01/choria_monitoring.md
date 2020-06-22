---
title: "Choria Monitoring"
date: 2020-07-01T09:00:00+01:00
tags: ["releases"]
draft: true
---

## Overview

I recently had to refresh my infrastructure and revisited how many things are done - I now host all the Choria 
components on Kubernetes using our WIP [helm repos](https://github.com/choria-io/helm) and so also had to revisit
monitoring for those components and look at what's up with my aging setup on the more traditional machines I host.

I did not find any appealing solution that's free and realised I had almost 100% of what I need to build a pretty
neat monitoring platform.  So I did just that and in this post and a few follow up ones I'll introduce the upcoming
platform features and what's next for them.

## Features

As Choria is very framework focused and really an operations integration platform at heart I am not trying to solve
all the problems myself but focusing on maximising the integration possibilities.  So here's first a few screenshots 
of Dashboards and such and then I'll break down the features.

![](health-overview-small.png)

This dashboard shows overall views of checks, problems and have an additional expanded view showing selected checks
and issues found.

![](machine-watch.png)

This is a life view of the Cloud Events being emitted by the system.

```puppet
choria::health_check{"check_typhon":
    plugin            => "/usr/lib64/nagios/plugins/check_procs",
    arguments         => "-C typhon -c 1:1",
    remediate_command => "service typhon restart",
}
```

Above creates a check which includes auto remediation.

Current main features:

 * Checks once configured runs without any central coordination and without requiring network connections
 * Supports the popular Nagios 4 exit code plugin API
 * Support auto remediation of issues without any central coordination
 * Choria RPC API to set maintenance mode for any check or force immediate checks, subject to AAA, 2FA and OPA Policies
   with support for authentication against Okta and other enterprise systems
 * Scalable to 100s of thousands of nodes
 * CLI observation tools to view live results stream
 * Can integrate with any other Autonomous Agent features such as cron like 5-field schedules for when checks should run
 * Alerting through the Prometheus Alert Manager with integrations to Pager Duty etc
 * Scalable to vast numbers of checks as there is no central coordination
 * Check results can be published to Prometheus Node Exporter
 * Prometheus + Grafana for over all health, aggregate statuses etc
 * Node level dashboards showing all checks for a node, alerts etc
 * For non Prometheus integrations each state transition gets published as a Cloud Events v1.0 format message
 * Each check result (or just problem ones) are published as Cloud Events v1.0 format messages
 * Optional configuration via Puppet
 
Future features:

 * Additional related Autonomous Agent Watchers like integration into goss
 * Check Configuration updated without Puppet utilising NATS JetStream
 * Purpose build CLI manager
 * Potentially a purpose specific distribution of the features into a purpose built binary
   * Single binary install
   * Auto enrolling into the Choria infrastructure
   * Auto enrolling into Certificate Authorities managed by Kubernetes Cert Manager
   * Suitable for deploying as side cars next to microservices in Kubernetes
   * Included event router to send Cloud Events to KNative and similar systems

## Conclusion

This is the first in a series of posts about this feature. The core work to add Nagios check support to the Autonomous
Agents took about 4 hours and all the rest of the features are essentially things Choria already had and that just
happened by default.

Follow up posts will look at aspects of the system in detail and explore how this was built.
