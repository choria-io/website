---
title: "External Python Agents"
date: 2019-09-23T9:09:00+01:00
tags: ["external", "python"]
draft: true
author: Ben Roberts
---

The latest release of Choria includes support for external agents, so now writing agents in any language is possible. For those of us who prefer to use Python to manage our systems, we have a new library, [py-mco-agent](https://github.com/optiz0r/py-mco-agent) which is intended to make it as simple as possible to write Choria agents. The simplest possible agent can be written with just a few lines of code.

## Shortest Python example

```python
#!/usr/bin/python3

from mco_agent import Agent, action, dispatch, register_actions


@register_actions
class Parrot(Agent):

    @action
    def echo(self):
        self.reply.data['message'] = self.request.data['message']


if __name__ == '__main__':
    # When run as a standalone executable, the dispatch method takes care of
    # processing the request and sending back the reply
    dispatch(Parrot)
```

This example creates a new agent named `parrot`, which supports a single action named `echo` (the action method name is used as the MCollective action, so only valid python identifiers are supported). All this action does is return the message given as input, but it demonstrates how to create an agent class, add an action, and run the dispatcher which takes care of processing the incoming request and calling the right action method.

The agent will also need the DDL files that describe the available actions. DDL files come in two formats presently; a JSON file which is read by the Choria server and used to dispatch actions to the python script, and a Ruby DDL file which is used by the `mco` client. Choria version 0.12.1+ includes a command-line wizard to generate the JSON DDL file, and to convert this (or an existing JSON DDL file) into the corresponding Ruby DDL file. To generate these, run `choria tool generate ddl parrot.json parrot.ddl` and answer the questions. The DDL files for the parrot example can be found in the `py-mco-agent` [examples](https://github.com/optiz0r/py-mco-agent/tree/master/examples).

## Handling inputs and outputs

The information in the request from Choria is made available to your agent in the `self.request` object. The [request fields](https://choria.io/docs/development/mcorpc/externalagents/#requests) are added as properties, including `self.request.data`, which is a dictionary containing all the action's input variables. Likewise, the Choria reply object is pre-created before your action is run, and made available as `self.reply` and has a `data` dict which will store your output variables. In the above example, there's a single `messages` input, and a `messages` output. The `echo` action simply copies the value of the input parameter to the output one and returns.

## Indicating failures

If your action needs to fail for any reason, you can do this by calling the `self.reply.fail(code, msg)` method. This takes two parameters, a numerical error code (which should match one of the pre-defined error codes listed in the table for the [request](https://choria.io/docs/development/mcorpc/externalagents/#requests)). The message may be any descriptive string.

```python
from mco_agent import Agent, action, register_actions

@register_actions
class Parrot(Agent):

    @action
    def echo(self):
        self.reply.fail(5, "I'm sorry, Dave, I'm afraid I can't do that")
```

## Conditional activation

By default the Python agent is activated on all hosts on which it is installed. You can control this behaviour if the agent should only be activated under certain circumstances, such as when additional dependencies are installed. The `Agent` base class defines a `should_activate` method which may return `True` if the agent should be activate, or `False` otherwise. It is called by Choria once on startup to determine if the node should be included in discovery. When an RPC request is received, the dispatcher calls this method a second time to ensure the agent should still be active before the action is executed.

```python
import os
from mco_agent import Agent, register_actions

@register_actions
class Parrot(Agent):

    @staticmethod
    def should_activate():
        return os.path.exists("/tmp/enable_parrot")
```

## Using config files

If you need to configure how your agent works, you can read the standard Choria `plugin.d` configuration files. The `Agent` class reads the configuration file matching the agent name on startup, and stores the contents within a dict-like `self.config` object. The file contains `key = value` lines, and settings can be read by looking up the key within the `self.config` object (which must exist or a `KeyError` exception will be raised). You can also use the familiar `get()` function, which will return a default value if the config setting was not found in the file or if the config file didn't exist at all.

```python
from mco_agent import Agent, action, register_actions

@register_actions
class Parrot(Agent):

    @action
    def echo(self):
        # get the "prefix" setting from the configuration file
        prefix = self.config.get('prefix', 'Polly says ')
        self.reply.data['message'] = prefix + self.request.data['message']
```

Configuration values easily be set using the puppet module config parameter.

## Using logging

The `dispatch` method takes care of setting up logging for you, and a pre-configured `self.logger` is available within your actions. Choria expects output on `stdout` to be informational log messages, and output on `stderr` to be error information. The standard Python `logging` library is used, with warning and higher messages mapped to `stderr`, and all lower messages mapped to `stdout`. The log level is set to `INFO` for your agent, with a logger hierarchy of `mcorpc.<agent_name>`. Logging for all other hierarchies is disabled by default to prevent unwanted output being sent back to Choria, but you can re-enable this if you wish by creating a new logger object for the desired hierarchy and calling `setLevel()` as needed.

```python
from mco_agent import Agent, action, register_actions

@register_actions
class Parrot(Agent):

    @action
    def echo(self):
        prefix = ""
        try:
            self.logger.info("Looking up prefix in the configuration file")
            prefix = self.config['prefix']
        except KeyError:
            self.logger.warning("No prefix found in the configuration file")

        self.reply.data['message'] = prefix + self.request.data['message']
```

## Installing the library

The [py-mco-agent](https://pypi.org/project/py-mco-agent/) library is available on pypi and can be installed using pip:

```bash
pip install py-mco-agent
```

or with Puppet:

```puppet
package {'py-mco-agent':
  ensure   => installed,
  provider => 'pip3',
}
```

It is written with Python 2.6+/3.6+ in mind, for compatibility with older operating systems (although the test suite does require Python 2.7+ to run).

## Installing the example Parrot agent

First up, copy the three files from the `py-mco-agent` [examples](https://github.com/optiz0r/py-mco-agent/tree/master/examples) directory to your mcollective agents directory (which is typically `/opt/puppetlabs/mcollective/plugins/mcollective/agent/`):

  * `parrot`
  * `parrot.json`
  * `parrot.ddl`

For the moment, RPC authorization in the Choria server is in beta and you must opt-in to enable this. If you do not, then anyone with access to the `choria` or `mco` clients will be able to run your agents. To enable this, enable the following setting in your Choria server configuration file (typically '/etc/choria/server.conf`):

```nohighlight
rpcauthorization = 1
```

Restart your `choria-server` service to pick up the config change and to load the new parrot agent

```nohighlight
# systemctl restart choria-server
```

Now you can try sending your first rpc request to the Python agent:

```nohighlight
$ choria req parrot echo message="hello world!"
Discovering nodes .... 1

1 / 1    0s [====================================================================] 100%

localhost.localdomain:
   Message: Parrot: hello world!


Finished processing 1 / 1 hosts in 393.793824ms
```

## Packaging your own agents using puppet modules

Choria's external agents including those using `py-mco-agent` can be packaged into puppet modules as described in the [packaging](https://choria.io/docs/development/mcorpc/packaging/#packaging) guide.

With the agent and DDL files inside an `agent` subdirectory, like this:

```nohighlight
parrot
└── agent
    ├── parrot
    ├── parrot.ddl
    └── parrot.json
```

We can run the following command to generate a puppet module suitable for uploading to the forge:

```bash
mco plugin package --vendor optiz0r
```

The module can then be installed onto your puppet server using `puppet module install`, or added to your `Puppetfile` if you use r10k, librarian-puppet or the PE code manager. Installing the agent on your nodes is as simple as listing the module in the `mcollective::plugin_classes` list in hiera.

## Other examples

* [puppet-env-manager-agent](https://github.com/optiz0r/puppet-env-manager-agent) - A simple agent that directly integrates the python library within [puppet-env-manager](https://github.com/optiz0r/puppet-env-manager) to update puppet directory environments on puppetservers using Choria from a CI/CD pipeline

## Issues and improvements

This Python library was written to help demonstrate the external request function in Choria and as such is fairly limited in functionality and not widely tested. If you find any issues or want to suggest any improvements, please feel free to contribute on the [py-mco-agent](https://github.com/optiz0r/py-mco-agent/) GitHub project.