+++
title = "Scout"
pre = "<b>5. </b>"
weight = 50
+++

## Introduction

Choria Scout is an effort to create a Monitoring Framework that builds on the capabilities of the Choria framework to 
create a modern, scalable and free cloud native monitoring system.

Scout is focused on maximal openness of data formats, licensing and packaging. Further, we try to make it as
easy as possible to integrate with 3rd party systems by choosing open data formats and easy extension points.

Scout was introduced in 2 initial blog posts:

 * [Introducing Choria Scout](https://choria.io/blog/post/2020/07/02/choria_scout/)
 * [Scout Components](https://choria.io/blog/post/2020/07/03/scout_components/)
 
Keep an eye out for [further blog posts](https://choria.io/blog/tags/scout/) related to Scout as we work on it.

## Status

Scout is a work in progress, today it features:

 * Framework level features in Choria Server that can run Nagios checks and perform remediation
 * Integration with [Goss](https://github.com/aelsabbahy/goss) for regular deep node inspection
 * Integration into the popular Prometheus system to publish check overview state
 * Publishes CNCF CloudEvents format messages about check statuses
 * Configurable using Puppet
 * Highly Scalable to 10s of thousands of nodes
 * Archives events in our Choria Streaming server
 
In the future we plan to release a full new distribution of Choria called *Choria Scout* that will be easy to deploy
and host on Kubernetes, self provisioning, self configuring and highly scalable. It will not rely on Puppet.
