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

Today in Puppet 5.x there is no concern everything continues to work, once Puppet 6 is released there will be a short delay until the Choria project is ready to run on Puppet 6.

We are working with Puppet Inc to give items like the code, mailing lists and more a new home, this page will get further updates as details become clear.

For further information please contact me am on the Puppet IRC as _Volcane_ and on slack as _ripienaar_, we also have a _#choria channel on Freenode of _#choria_ on the [Puppet Slack](http://slack.puppet.com/).

## Source Code

Source code for legacy Marionette Collective projects was donated to Choria by Puppet Inc, archival copies of the code is in the [Choria Legacy](https://github.com/choria-legacy) project.  Of these legacy projects a number of plugins are being maintained and evolved, see the [Choria Plugins](https://github.com/choria-plugins/) project for those.

## Timeline

|Date|Event|
|----|-----|
|2018/04/24|[Preview of The Choria Server](/docs/configuration/choria_server/) released that replaces _mcollectived_|
|2018/07/15|Key documents from the _mcollective_ project adopted detailing usage of the RPC framework|
|2018/07/17|Official deprecation by Puppet Inc|
|2018/07/25|#choria created on Freenode|
|Fall 2018|Target date for Puppet 6 and removal of _mcollective_ libraries from _puppet-agent_|
|2018/11/19|Puppet Inc officially hand over all _mcollective_ related code, hosted in [Choria Legacy](https://github.com/choria-legacy)|
|2018/12/01|Initial Choria releases that support Puppet 6|
