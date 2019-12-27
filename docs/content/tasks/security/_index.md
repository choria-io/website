+++
title = "Security"
weight = 300
+++

Choria integrates Puppet Tasks tightly with its AAA (Authentication, Authorization and Auditing) suite, this section will introduce you to the features of each.

## Authorization

Any Choria authorization plugin will be supported by the Tasks integration.  Of course Choria configures only the Action Policy so the documentation will focus on that.

The agent have a number of actions, you generally will give people access to all of these:

|Action|Description|
|------|-----------|
|download|Downloads a Puppet Task into a local cache|
|run\_and\_wait|Runs a Puppet Task that was previously downloaded, wait for it to finish|
|run\_no\_wait|Runs a Puppet Task that was previously downloaded do not wait for it to finish|
|task\_status|Request the status of a previously run task|

If you give someone access to _download_, _run\_and\_wait_ and _run\_no\_wait_ they can initiate and run tasks, you can give someone access to _task\_status_ only to view statuses.

Note that the data plugin effectively shows all the _task\_status_ action shows and there are no RBAC for those.  So basically _task\_status_ is always open.

So the example from earlier in this document gives _choria=rip.mcollective_ full access to the Puppet Task feature:

```yaml
mcollective_agent_bolt_tasks::policies:
  - action: "allow"
    callers: "choria=rip.mcollective"
    actions: "download run_and_wait run_no_wait task_status"
    facts: "*"
    classes: "*"
```

Once you have this in place you have to authorize specific tasks:

```yaml
mcollective_agent_bolt_tasks::policies:
  - action: "allow"
    callers: "choria=rip.mcollective"
    actions: "puppet_conf gcompute::snapshot"
    facts: "*"
    classes: "*"
```

Here this user will have the ability to run the _puppet\_conf_ task as well as the _gcompute::snapshot_ one but no others.

Every task invocation will call the RBAC system twice - one for the _run\_and\_wait_ or _run\_no\_wait_ action and once for an action matching the task name.

## Auditing

Auditing is done as per normal Choria Audit Logs and are written in JSON format.  For every _run_ there would be 2 actions performed - one of the run variants and one for the actual task name being invoked.

```nohighlight
$ sudo tail -n 1000 /var/log/puppetlabs/mcollective-audit.log|jq 'select(.request_id=="45250c07824f5922be68468d08f6b76c")'
{
  "timestamp": "2018-03-19T13:50:51.762983+0000",
  "request_id": "45250c07824f5922be68468d08f6b76c",
  "request_time": 1521467451,
  "caller": "choria=rip.mcollective",
  "sender": "dev1.devco.net",
  "agent": "bolt_task",
  "action": "run_and_wait",
  "data": {
    "task": "puppet_conf",
    "files": "[{\"filename\":\"init.rb\",\"sha256\":\"38351bb4b72a29064f5c9a224a81f1abf7042b0bc9d1ffd2a074d1bd63b2f246\",\"size_bytes\":1231,\"uri\":{\"path\":\"/puppet/v3/file_content/tasks/puppet_conf/init.rb\",\"params\":{\"environment\":\"production\"}}}]",
    "input": "{\"action\":\"set\",\"section\":\"user\",\"setting\":\"modulepath\",\"value\":\"/tmp/modules\"}",
    "process_results": true
  }
}
{
  "timestamp": "2018-03-19T13:50:51.763933+0000",
  "request_id": "45250c07824f5922be68468d08f6b76c",
  "request_time": 1521467451,
  "caller": "choria=rip.mcollective",
  "sender": "dev1.devco.net",
  "agent": "bolt_task",
  "action": "puppet_conf",
  "data": {
    "task": "puppet_conf",
    "files": "[{\"filename\":\"init.rb\",\"sha256\":\"38351bb4b72a29064f5c9a224a81f1abf7042b0bc9d1ffd2a074d1bd63b2f246\",\"size_bytes\":1231,\"uri\":{\"path\":\"/puppet/v3/file_content/tasks/puppet_conf/init.rb\",\"params\":{\"environment\":\"production\"}}}]",
    "input": "{\"action\":\"set\",\"section\":\"user\",\"setting\":\"modulepath\",\"value\":\"/tmp/modules\"}",
    "process_results": true
  }
}
```

Above you can see the double RBAC in action - once for _action_ _run\_and\_wait_ and once for _puppet\_conf_.

You might of course find additional status calls etc too:

```nohighlight
$ sudo tail -n 1000 /var/log/puppetlabs/mcollective-audit.log|jq 'select(.data.task_id=="45250c07824f5922be68468d08f6b76c")'
{
  "timestamp": "2018-03-19T13:51:41.232263+0000",
  "request_id": "3beaa976857f527281ac192602acad83",
  "request_time": 1521467501,
  "caller": "choria=rip.mcollective",
  "sender": "dev1.devco.net",
  "agent": "bolt_task",
  "action": "task_status",
  "data": {
    "task_id": "45250c07824f5922be68468d08f6b76c",
    "process_results": true
  }
}
```
