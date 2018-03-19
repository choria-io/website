+++
title = "Security"
weight = 300
+++

Choria integrate Puppet Tasks rightly with it's AAA (Authentication, Authorization and Auditing) suite, this section will introduce you to the features of each.

## Authorization

Any MCollective authorization plugin will be supported by the Tasks integration.  Of course Choria configures only the Action Policy so the documentation will focus on that.

The agent have a number of actions, you generally will give people access to all of these:

|Action|Description|
|------|-----------|
|download|Downloads a Puppet Task into a local cache|
|run_and_wait|Runs a Puppet Task that was previously downloaded, wait for it to finish|
|run_no_wait|Runs a Puppet Task that was previously downloaded do not wait for it to finish|
|task_status|Request the status of a previously ran task|

If you give someone access to _download_, _run_and_wait_ and _run_no_wait_ they can initiate and run tasks, you can give someone access to _task_status_ only to view statusses.

Note that the data plugin effecitvely shows all the _task_status_ action shows and there are no RBAC for those.  So basically _task_status_ is always open.

So the example from earlier in this document gives _choria=rip.mcollective_ full access to the Puppet Task feature:

```yaml
mcollective_agent_bolt_tasks::policies:
  - action: "allow"
    callers: "choria=rip.mcollective"
    actions: "download,run_and_wait,run_no_wait,task_status"
    facts: "*"
    classes: "*"
```

## Auditing

Auditing is done as per normal Choria Audit Logs and are written in JSON format.  For every _run_ there would be 2 actions performed - download and one of the run variants.

<pre><code class="nohighlight">
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
</code></pre>

You might of course find additional status calls etc too:

<pre><code class="nohighlight">
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
</code></pre>
