---
title: "Statement on Puppet Support"
date: 2025-01-23T00:00:00+01:00
tags: ["releases", "puppet"]
draft: false
---

There has been a significant change in the Open Source landscape of Puppet in recent months, I will not go too much into
the details but suffice to say that Perforce (Puppet trademark owner) have essentially created a private closed-source
fork of Puppet and from now on if you get "puppet" packages they are this rogue anti-community closed source fork.

Some relevant blog posts:

 * [Our Plans for Open Source Puppet in 2025](https://www.puppet.com/blog/open-source-puppet-updates-2025)
 * [It was only a matter of time](https://overlookinfratech.com/2024/11/08/sequestered-source/)

At this point, any official Puppet release is a high-risk, closed source fork, I do not suggest anyone keep using these
packages as you are essentially running closed source disguised as Open Source. You will not be able to see git history
of what you are running, you will not be able to see build tooling or related infrastructure.

This left the community in a lurch, but luckily the good people from Vox Pupuli has come to the rescue and now hosts the
only actually Open Source implementation of the Puppet Specification called [Open Vox](https://voxpupuli.org/openvox/).
There has also been significant build pipeline work which is now entirely open and visible to everyone - a significant
strengthening of the Open Source commitment.

Some relevant blog posts:

 * [First release, hot off the presses!](https://voxpupuli.org/blog/2025/01/21/openvox-release/)
 * [First release, hot off the presses!](https://overlookinfratech.com/2025/01/21/first-release-hot-off-the-presses/)

Going forward there will be no way to test Choria against the rogue fork of Puppet that Perforce maintains without signing
an EULA (as yet unseen by anyone). There will also be limits to how many open source servers one may manage (25).

As such, Choria will not continue to support Puppet as released by Perforce and will take steps in future releases to require
Open Vox.

Read on for some timelines and release details.

<!--more-->
The next release of Choria Server will be `0.30.0` and that will see some updates to the Puppet modules also, I will leave 
that as is and continue to support Puppet as it stands. There would be only the most essential bug fixes in that release 
series.

Shortly after I'll release `0.31.0` which would be nearly identical to `0.30.0` but with steps taken in the modules to
fail when running against the rogue closed source fork.
