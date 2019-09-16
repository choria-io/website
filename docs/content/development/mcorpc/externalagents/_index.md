+++
title = "External Agents"
weight = 21
+++

As of version *0.12.1* of the Choria Server we support something called `External Agents`, these let you write agents hosted by the Choria Server in any language.  These agents are forked on demand and receive the request in a temporary file and write their reply in a temporary file as JSON.

{{% notice tip %}}
This is a brand new feature and subject to change and quite possibly have some bugs!
{{% /notice %}}

## Overview

### File Layout

An external agent can be written in any language and the related files go in the same directory as other Ruby agents, here is a basic *helloworld* agent on disk in `/opt/puppetlabs/mcollective/plugins/mcollective/agent`:

```nohighlight
-rwxr-xr-x 1 root root 315 Sep 12 11:18 helloworld
-rw-r--r-- 1 root root 819 Sep 12 10:40 helloworld.ddl
-rw-r--r-- 1 root root 915 Sep 12 10:40 helloworld.json
```

Both the [DDL](../ddl/) files are required and can be generated using the new `choria tool generate ddl helloworld.json helloworld.ddl` utility.

### Communication Protocols

Communication from the Choria Server to your agent is done via files on disk and environment variables.  When run these variables will be set:

|Variable|Meaning|
|--------|-------|
|CHORIA_EXTERNAL_REQUEST|The path to where the JSON request data is found|
|CHORIA_EXTERNAL_REPLY|The path where your agent should write JSON reply data|
|CHORIA_EXTERNAL_PROTOCOL|Indicating if this is a request (choria:mcorpc:external_request:1) or activation (choria:mcorpc:external_activation_check:1) message|

Your agent will also be called with 3 arguments:

```
helloworld <request file> <reply file> <protocol>
```

The working directory will be your OS temporary directory.

### Logging

Any output your agent produce on *STDOUT* is logged at *INFO* level on the server.  Any *STDERR* output is logged as *ERROR* level on the server.

## Activation

At server start your agent will be called with a activation check, this gives you the chance to verify your dependencies and decide if the agent should be active on the particular node.

The `CHORIA_EXTERNAL_PROTOCOL` will be set to `choria:mcorpc:external_activation_check:1`, you should only respond to activation checks when this is set.

The `CHORIA_EXTERNAL_REQUEST` file will look like this:

```json
{
  "protocol": "choria:mcorpc:external_activation_check:1",
  "agent": "helloworld"
}
```

And you should reply with this in the `CHORIA_EXTERNAL_REPLY`:

```json
{
  "activate": true
}
```

Any reply not in this format or non 0 exit code from your code will result in the agent not being activated on this node. Your program will have maximum of 2 seconds to process an activation check.

## Requests

Once activated any requests sent to your agent will result in `CHORIA_EXTERNAL_PROTOCOL` set to `choria:mcorpc:external_request:1`.

The `CHORIA_EXTERNAL_REQUEST` file will look like this:

```json
{
 "protocol": "choria:mcorpc:external_request:1",
 "agent": "helloworld",
 "action": "ping",
 "requestid": "034c527089f746248822ada8a145f499",
 "senderid": "dev1.devco.net",
 "callerid": "choria=rip.mcollective",
 "collective": "mcollective",
 "ttl": 60,
 "msgtime": 1568281519,
 "body": {
   "agent": "helloworld",
   "action": "ping",
   "data": {
     "msg": "hello"
   }
 }
}
```

Few things to note here:

 * *requestid* is unique per request and will appear in audit logs and elsewhere
 * *callerid* will have been verified by the security system so you can rely on this being valid
 * *msgtime* is seconds since 1970 in UTC timezone
 * *body.data* is free form data, typically key=val of whatever the client sent
 * *body.data* will have been verified subject to the DDL validations, data types and default values set as appropriate

Your reply should be written to `CHORIA_EXTERNAL_REPLY` and must look like this:

```json
{
  "statuscode": 0,
  "statusmsg": "OK",
  "data": {
    "result": "hello"
  }
}
```

The *statuscode* is standard MCollective Protocol status codes:

|Status Code|Description|Exception Class|
|-----------|-----------|---------------|
|0|OK| |
|1|OK, failed.  All the data parsed ok, we have a action matching the request but the requested action could not be completed.|RPCAborted|
|2|Unknown action|UnknownRPCAction|
|3|Missing data|MissingRPCData|
|4|Invalid data|InvalidRPCData|
|5|Other error|UnknownRPCError|

And *data* is free form data being sent back to the client.  Note however the data too will be validated against the DDL output section, defaults will be set and more.

## Helper Libraries

We hope to see helper libraries for various languages from the community, we'll maintain a list of some here:

 * [Python](https://github.com/optiz0r/py-mco-agent) - Contributed by Ben Roberts as part of working on the early POC