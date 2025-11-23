+++
title = "The Marionette Collective Deprecation"
hidden = true
chapter = false
+++

## Background

On the 17th of July 2018 Puppet Inc officially announced that as of release 5.5.4 of the Puppet Agent The Marionette Collective has been deprecated.

The Choria project provides a path forward for users who wish to keep using MCollective and we are actively developing new technologies in this space.  Please review the [Choria Website](https://choria.io) for further information.

## Details

We welcome users of The Marionette Collective who wish to continue using it, we have gone to great length to ensure you can continue as always - and the Choria project added a lot of new and exciting features to the platform.

While the Choria project builds heavily on top of the foundation laid by The Marionnette Collective this does not mean an end or even a significant hurdle for this project.

Today in Puppet 5.x there is no concern everything continues to work, Puppet 6 users are automatically updated to the [Choria Server](/docs/configuration/choria_server/), and things mostly keep working.  Though review that page for status.

Puppet Inc have donated all the legacy MCollective code, this code is hosted in [Choria Legacy](https://github.com/choria-legacy) and we have made compatibility layers to allow the `mco` CLI to continue working with the Choria Server using this donated code.

Choria - and MCollective via the compatability layer - is in wide use by the community, we have some work left to do but the transition is generally smooth now.

For further information please contact me am on the Puppet IRC as _Volcane_ and on slack as _ripienaar_, _#choria_ on the [Vox Pupuli Slack](https://short.voxpupu.li/puppetcommunity_slack_signup) and [GitHub Discussions](https://github.com/choria-io/general/discussions).

## Source Code

Source code for legacy Marionette Collective projects was donated to Choria by Puppet Inc, archival copies of the code is in the [Choria Legacy](https://github.com/choria-legacy) project.  Of these legacy projects a number of plugins are being maintained and evolved, see the [Choria Plugins](https://github.com/choria-plugins/) project for those.
