+++
title = "Autonomous Agents"
pre = "<b>5. </b>"
weight = 50
+++

Autonomous Agents allow you to create agents that can run on any node continuously without the need to initiate actions via RPC calls.

{{% notice tip %}}
This feature is available since *Choria Server 0.11.0*
{{% /notice %}}

{{% notice warning %}}
This is a preview level feature, we're including to seek community feedback on an MVP level feature set
{{% /notice %}}

## Use Cases

These agents are designed to run continuously for the life of the Choria Server where they manage the node or environment the node runs in.

These are not designed to replace entire systems like Puppet, below we have some typical use cases, however it is designed to integrate with any other system via a simple shell + exit code interface.

### HVAC

You could have systems that monitor the air quality of a room and an air conditioner to improve the quality of the air in the room.  You do not wish to run these continuously, perhaps you built them using a few Raspberry PI.

You'd have a sensor or set of sensors that monitor the air quality and a network enabled switch to start and stop your HVAC.

An Autonomous Agent could be created to continuously query the air quality and turn the HVAC on and off on demand. Integration with the Raspbery PI systems would be via scripts you supply.

### Cron with event input

You have to gather data about the state of the machine, in general you wish to gather the data once every hour. However should a file or set of files get updated you want to run an immediate gather.

An Autonomous Agent could watch these files and execute the command every 4 hours.

### Container Management

You have a manifest that describes a container and its desired tag:

```json
{
    "image": "acme/sample",
    "tag": "123"
}
```

You want the system to continuously monitor this file and:

 * Watch for changes to the file
 * Trigger a deploy of the container at this version if that is not what's running
 * Continuously monitor the health of the container
 * Remediate the container by restarting it should the health check fail
 * Should the manifest change at any time, redeploy the container to the new desired version

 An Autonomous Agent could describe these interactions and the integration with docker could be shell scripts, ruby scripts or anything else like Ansible.

### Cluster Management

A manager host in a small cluster could health check the entire cluster and perhaps even own scaling the cluster to desired states.  The various health checks, upgrade and downgrade flows and remediation could all be implemented as Ansible Playbooks or Bolt Plans using the Autonomous Agent simply as a scheduler and coordinator between these tools.

### Monitoring

Using the included `nagios` watcher one can run Nagios compatible plugins:

 * Run them on a schedule
 * Perform auto remediation when in specific states
 * Consume `cloudevents` about check states 
 * Expose monitoring state to Prometheus via the included `node_exporter` integration for alerting and graphing
