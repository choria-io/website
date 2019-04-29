+++
title = "Watcher Reference"
weight = 30
toc = true
+++

## Common watcher properties

All watchers share a common set of properties detailed below, watcher specific properties are always in the *properties* key:

|Property          |Required          |Description|
|------------------|------------------|-----------|
|name              |:white_check_mark:|A unique name for the watcher|
|type              |:white_check_mark:|A known type of watcher like *file* or *exec*|
|state_match       |:white_check_mark:|A list of state names where this watcher is valid for|
|fail_transition   |                  |If set this event fires on failure|
|success_transition|                  |If set this event fires on success|
|interval          |                  |Runs the watcher every interval, valid intervals are of the form *1s*, *1m*, *1h*|
|announce_interval |                  |Announce the current state of the watcher regularly, valid intervals are of the form *1s*, *1m*, *1h*|
|properties        |:white_check_mark:|Watcher specific settings|

## File watcher

The *file* watcher observes a specific file for changes and presence. Today only a basic *mtime* check is done, in time other dimensions like hashes or even *inotify* based observation will be supported.

### Properties

|Property            |Required                  |Description|
|--------------------|--------------------------|-----------|
|path                |:white_check_mark:        |The path to the file to watch relative to the watcher manifest directory|
|gather_initial_state|                          |Gathers the initial file mode, stats etc for regular announces but only perform first watch after *interval*|

### Behavior

A file watcher will at *interval* times do a *mtime* check on the file.

If the file is missing a *fail_transition* event fires and an announcement is made.

If the file has changed since the previous run a *success_transition* event fires and an announcement is made. This means the first check would set the initial state after which changes are detected.  You could by setting *gather_initial_state* have the system gather initial file state on startup so the first regular watch would detect a change.

If the file have not changed nothing is published on every check, however a regular state announce can be done by setting *announce_interval*.

## Exec watcher

The *exec* watcher supports running shell commands, it has a very basic exit code based interface, output from the commands is not significant.

### Properties

|Property            |Required                  |Description|
|--------------------|--------------------------|-----------|
|command             |:white_check_mark:        |The command to run relative to the watcher manifest directory|

### Behavior

An exec watcher will at *interval* times run the command specified with a few machine specific environment variables set.

|Variable              |Description|
|----------------------|-----------|
|*NACHINE_WATCHER_NAME*|The *name* of the watcher being run|
|*NACHINE_NAME*        |The *name* of the machine being run|

The command is run with current directory set to the directory where the *machine.yaml* is, when the command exits *0* a *success_transition* fires, when it exits *!0* a *fail_transition* fires. Both cases publish an event announcing the execution.