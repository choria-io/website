+++
title = "Reports"
weight = 450
toc = true
+++

When running a playbook a report is produced that include things like all the inputs, every node set and statusses for every task.

When running on the CLI you can save this report to a local YAML file:

```bash
$ mco playbook run --report playbook.yaml
```

This will write a YAML file with a report with the following data:

## report

Basic report metadata

|Field|Type|Description|
|-----|----|-----------|
|version|Integer|The report format version, currently version 1|
|timestamp|Date|UTC time when this report was produced|
|success|Boolean|The overall playbook outcome|
|fail_message|String|In the event of a unhandled exception you'll get some message here|

## playbook

Basic playbook metadata

|Field|Type|Description|
|-----|----|-----------|
|name|String|The playbook name from the metadata|
|version|String|The playbook version from the metadata|

## inputs

|Field|Type|Description|
|-----|----|-----------|
|static|Hash|Hash of statically supplied inputs|
|dynamic|Hash|List of dynamic inputs|

Currently *dynamic* inputs are not supported by the playbooks.

Static inputs are listed as a hash with the key being the input name and value the input value

## nodes

Nodes is a hash where the key is the name of a node set and the value is a list of the resolved node names

## tasks

A report for every task will be listed here, each report has the following fields

|Field|Type|Description|
|-----|----|-----------|
|type|String|The task type|
|set|String|The task set the task was ran in|
|description|String|The task description|
|start_time|Date|Timestamp when the task was started in UTC time|
|end_time|Date|Timestamp when the task was stopped in UTC time|
|run_time|Float|How long the task ran|
|ran|Boolean|If the task have been executed, probably always true if its here|
|msg|String|The output message from the task|
|success|Boolean|If the task passed or failed|

## metrics

Overall metrics for the playbook run with totals by type of task etc

|Field|Type|Description|
|-----|----|-----------|
|start_time|Date|Same as report timestamp|
|end_time|Date|When the report was finalized|
|run_time|Float|Total run time for the playbook|
|task_count|Integer|Number of tasks that were ran|
|task_types|Hash|Metrics per type of task|

For every task type a entry in the *task_types* hash exist with these keys:

|Field|Type|Description|
|-----|----|-----------|
|count|Integer|How many tasks of a specific type was ran|
|total_time|Float|Time taken for all tasks of this type|
|pass|Integer|How many of this kind of task passed|
|fail|Integer|How many of this kind of task failed|

## logs

Every log that would have been shown - dependant on the playbook log level - are included here, each log has these keys:

|Field|Type|Description|
|-----|----|-----------|
|time|Date|UTC time the log was produced|
|level|String|The log level|
|from|String|caller info for which ruby class and method produced the log line|
|msg|String|The message body
