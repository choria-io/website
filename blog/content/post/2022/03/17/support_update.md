---
title: "Supported OS and Go versions update"
date: 2022-03-17T00:00:00+01:00
tags: ["releases"]
draft: false
---

Just a small headsup to alert users of a few changes in the Operating Systems we support.

## Operating Systems

We support Operating System packages primarily for our Puppet users, and, so we tend to follow along with their deprecations.

Yesterday Puppet announced they will drop support for Ubuntu Xenial which prompted me to do the same here. Xenial has been
a long and painful distribution to support, I am eager to not have to deal with it anymore. At the same time we are
removing support for Debian Stretch and EL6 (though we have not done packages for EL6 for a long time). We will not build
packagers for these Operating Systems and future releases will not have any RPMs or DEBs published for them.

We will in the near term future archive the repositories that was used to serve these Operating Systems.

## Golang

The Golang team announced Go 1.18 this week, in line with Go support policies we therefore also dropped support for
Go 1.16 and made some breaking changes that would prevent Go 1.16 from compiling the code base. Nightly Choria builds
are already done using 1.18 to give us ample time to test the new compiler.

These updates will slowly ripple through Puppet modules and other related projects also.
