---
title: "Docker Changes"
date: 2023-03-20T00:00:00+01:00
tags: ["releases"]
draft: false
---

We used publish our releases to Docker Hub for some time now, we use those with Docker Compose demos and our Helm charts.

Docker have over the years been clawing back their free tiers, first starting to delete old containers which killed off
many of our old artifacts and now entirely stopping the free teams tier.

We thus had to make alternate plans. We approached [Container Registry](https://container-registry.com/) about their
dedicated plans, and they graciously offered us a handsome discount over their usual fees which we gladly accepted.

Container Registry is run by maintainers of the [Harbor Project](https://www.cncf.io/projects/harbor/), so I
am happy to support them in their efforts.

A significant feature of their offering is the ability to host our registry on a custom domain, this means should we have
to move again in future we will hopefully do so with a smaller impact on our users as we can take our name and paths with.

As of today, 2023-03-20, we publish all our containers to our new registry at `registry.choria.io` and will soon move our
helm charts to this registry as well as it's a full OCI registry.

Today we publish the following releases:

 * `registry.choria.io/choria/choria` [Choria Broker and Server](https://github.com/choria-io/go-choria)
 * `registry.choria.io/choria/stream-replicator` [Choria Stream Replicator](https://github.com/choria-io/stream-replicator)
 * `registry.choria.io/choria/provisioner` [Choria Server Provisioner](https://github.com/choria-io/provisioner)
 * `registry.choria.io/choria/aaasvc` [Choria AAA Service](https://github.com/choria-io/aaasvc)
 * `registry.choria.io/choria/asyncjobs` [Choria Asynchronous Jobs](https://github.com/choria-io/asyncjobs)
 * `registry.choria.io/choria/packager` Packaging tooling

We also publish Nightly builds - and these now are all automatically build and published every day and retained for around 30 days:

 * `registry.choria.io/choria-nightly/choria`
 * `registry.choria.io/choria-nightly/aaasvc`
 * `registry.choria.io/choria-nightly/provisioner`
 * `registry.choria.io/choria-nightly/stream-replicator`

Once again huge shout out to [Container Registry](https://container-registry.com/) for the discounted offer.