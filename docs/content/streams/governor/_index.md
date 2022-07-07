+++
title = "Governor"
weight = 20
+++

Choria Governor allows you to control number of concurrent executions and, optionally, number of executions per period
of time.

These 2 abilities allow us to solve complex problems faced in making Cron job execution reliable namely preventing all
the cron jobs from running at the same time and making cron servers reliable by spreading execution over a pool of servers
while only running a job on one server.

## Managing overall concurrency

When running tasks on network servers that access shared resources like databases, Puppet Servers or other similarly
constrained resource services its desirable to limit the concurrency of executions.

Many approaches exist, for example, spreading the cron minute flag out using random distribution. This works ok, but it
can be hit or miss and does not solve the problem for adhoc executions.

Choria Governor lets you use Choria Streams to constrain the concurrency of network tasks, one could run a cron command
fleet wide at the same time and use the Governor to limit how many concurrent processes will execute.

### Creating a Governor

With Choria Streams configured you can create a Governor using the CLI:

```nohighlight
$ choria governor add PUPPET 10 20m 3

  Capacity: 10
   Expires: 20m0s
  Replicas: 3
```

This creates a Governor called *PUPPET* that will allow up to 10 concurrent processes and that is reliably distributed
across 3 Choria Brokers.

The expires is a fail-safe, if the process crashes halfway through execution and does not remove itself from the Governor
then that entry will stay, if this happens a lot the Governor could be full of orphaned leases and no further work can be done.

The expires setting - 20 minutes here - will expire entries that's not been cleaned up by the calling process. Care should
be taken to choose an appropriately safe setting.

Running this command again with different settings will edit the Governor, adjusting the capacity will immediately impact
the running fleet with no restarts or update of the fleet nodes needed. The Replicas setting can not be edited.

As of version 0.26.0 of the `choria/choria` module, you can also use Puppet to manage Governors:

```puppet
choria_governor {"PUPPET":
    capacity => 10,
    expire   => 20*60,
    replicas => 3,
}
```

When managing the Governor used to limit Puppet runs there is a chicken and egg, so perhaps do not run your Puppet Servers
against a Governor but on a traditional schedule and let them manage the Governors.

Replicas is not changeable without destroying the Governor, so if you created with Replica 1 and want to move to 3 you
will need to set `force => true` on the `choria_governor` resource which will then destroy and recreate the Governor.

### Executing a CRON job

To use this Governor to control a cron job use the `choria governor run` command:

```nohighlight
*/30 * * * * /bin/choria governor run PUPPET \
             --config /etc/choria/server.conf \
             --max-wait 20m \
             '/opt/puppetlabs/bin/puppet agent --onetime --no-daemonize --no-usecacheonfailure --no-splay'
```

Here we run `puppet agent` controlled by the Governor `PUPPET` and the `--max-wait` setting means the CLI will keep
trying to get a slot for 20 minutes, if it fails it will stop trying. Use this setting to prevent overlapping executions.

This can be used in shell scripts, CLI or just anywhere you need to limit concurrency. The CLI will make a connection
to the Choria Broker and so needs certificates etc, here we use the `server.conf` to use the server certificates. Keep
in mind that each process makes a connection to the broker and that Choria Broker is limited to 50 000 concurrent connections
by default.

## Limiting number of runs per period

In this mode we might have a pool of cron servers running jobs but only one of them should run a job. Others will simply
exit with 0 exit code in that scenario. We might also say a specific job can only run 5 times per period etc.

This add reliability to cron execution in that machines can fail but jobs continue to run.

This particular mode is good for isolated jobs, for a set of related jobs see our section on leader election.

{{% notice tip %}}
This feature added in Choria v0.26.1
{{% /notice %}}

### Creating a Governor

With Choria Streams configured you can create a Governor using the CLI:

```nohighlight
$ choria governor add ACCOUNTING 1 29m 3

Capacity: 1
Expires: 29m0s
Replicas: 3
```

Like the previous example we first create a governor for our Cronjob, here we allow only 1 execution and we set the expiry
to 29 minutes.  The period is chosen since will run our cron every 30 minutes so this should be sufficient to control it.

### Executing a CRON job

To use this Governor to control a cron job use the `choria governor run` command:

```nohighlight
*/30 * * * * /bin/choria governor run ACCOUNTING \
             --config /etc/choria/server.conf \
             --max-wait 10s \
             -max-per-period \
             '/usr/local/bin/account.sh'
```

We pass the new flag `-max-per-period` that enables the mode where we limit executions per Governor expiry period.

That's it, this cron job can now run on 10 machines and every 30 minutes one of the 30 will run the task.

## Controlling Choria Autonomous Agent

The Choria Autonomous Agent `exec` Watcher supports Governors, here we use the *NATS_SUPERCLUSTER* Governor to control rolling updates
of a service:

```yaml
  type: exec
  interval: 20s
  success_transition: health_check
  state_match:
  - REGION_UPDATE
  properties:
    command: "/srv/nats/bin/update.sh"
    timeout: 60s
    governor: NATS_SUPERCLUSTER
```

Executing this will NOT make additional connections to the network, the connection the Choria Server maintains permanently
will be reused.

## Managing the Governor

During execution each node will register themselves with the Governor and remove themselves at the end of the run, you
can view current in use slots:

```nohighlight
$ choria governor view PUPPET
Configuration for Governor PUPPET

       Capacity: 5
        Expires: 20m0s
       Replicas: 3
  Active Leases: 3


+------+-------------------+---------+
|  ID  |  PROCESS NAME     |   AGE   |
+------+-------------------+---------+
| 2412 | dev12.example.net | 29.597s |
| 2413 | dev10.example.net | 15.26s  |
| 2414 | dev20.example.net | 10.966s |
+------+-------------------+---------+
```

Here we see that each least has a ID, these IDs will always increase and not wrap down to zero. If you wish to evict
a certain entry from the Governor you can do so using `choria governor evict PUPPET 2412`.

Should the entire Governor be filled with orphan entries and you do not wish to wait for expire you can run `choria
governor reset PUPPET` which will delete all entries.

Finally if not needed anymore the Governor can be removed using `choria governor rm PUPPET`.

## Observing runs

During a run the CLI or Autonomous Agent will publish Choria Lifecycle Events that can be viewed using `choria tool event`

```nohighlight
$ choria tool event --type governor
09:32:55 [governor] dev13.example.net: obtained slot 2421 on PUPPET
09:33:10 [governor] dev8.example.net: obtained slot 2422 on PUPPET
09:33:15 [governor] dev17.example.net: vacated slot 2420 on PUPPET
09:33:16 [governor] dev5.example.net: obtained slot 2423 on PUPPET
09:33:17 [governor] dev13.example.net: vacated slot 2421 on PUPPET
09:33:19 [governor] dev18.example.net: obtained slot 2424 on PUPPET
09:33:33 [governor] dev8.example.net: vacated slot 2422 on PUPPET
```
