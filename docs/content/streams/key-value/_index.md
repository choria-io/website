+++
title = "Key-Value Store"
weight = 30
+++

Key-Value stores like *etcd* and *consul* have become corner stones of modern infrastructure. In short, they are simple
schemaless stores where you can store any bytes in a bucket given a single key name. The values can later be retrieved.

Choria Key-Value Store is built on Choria Streams and share many of the features plus some KV specific:

 * Replicated, share-nothing storage
 * Possibility to make regional read-replicas
 * History kept per key
 * Non destructive deletes
 * Ability to watch a key or entire bucket for changes
 * Fast in-memory read caches
 * Integrated with [Choria Autonomous Agent](../autoagents)
 * Client libraries for Go with more to follow

{{% notice tip %}}
This feature added in Choria v0.23.0
{{% /notice %}}

## Managing and Using a Bucket
### Adding a bucket

To store data you have to add a bucket, with Choria Streams enabled this is easy:

```nohighlight
$ choria kv add CFG \
     --history 5 \
     --replicas 3 \
     --max-value-size 10240 \
     --max-bucket-size 102400
```

Above we add a bucket *CFG*, it keeps 5 historic values for any key, the data
is replicated across 3 Choria Brokers, and some limits are set for sizes.

### Writing a value

Any data can be stored in the bucket:

```nohighlight
$ choria kv put CFG username myservice
myservice
```

```nohighlight
$ cat cfg.json|choria kv put CFG config -- -
```

### Getting a value

You can get the value with a bit of metadata shown:

```nohighlight
$ choria kv get CFG username
CFG > username created @ 13 Jul 21 10:38 UTC

myservice
```

But we can also use it in a script friendly manner:

```bash
USER=$(choria kv get CFG username --raw)

echo "Username: ${USER}"
```

### Deleting a key

Deletes are non-destructive, meaning the past data will be kept if history is configured:

```noghighlight
$ choria kv del CFG username
? Really remove the username key from bucket CFG (y/N)
$ choria kv get CFG username
FATA[0000] Could not run Choria: unknown key
```

You can pass `-f` to not be prompted

### Viewing history for a key

Above we configure 5 value history for the bucket, so we can see the historic values:

```nohighlight
$ choria kv history CFG username
+-----+-----------+----------------------+--------+-----------+
| SEQ | OPERATION | TIME                 | LENGTH | VALUE     |
+-----+-----------+----------------------+--------+-----------+
| 1   | PUT       | 13 Jul 21 12:38 CEST | 9      | myservice |
| 2   | DEL       | 13 Jul 21 12:40 CEST | 0      |           |
+-----+-----------+----------------------+--------+-----------+
```

Here we can see the *DEL* operation was added which represents the delete.

### Watching a bucket or key

One can watch writes into a bucket in real time, this can be for the entire bucket which will show full history.

```nohighlight
$ choria kv watch CFG username
[2021-07-13 12:38:10] PUT CFG.username: myservice
[2021-07-13 12:40:07] DEL CFG.username
```

There's a special form of watch that is like a waiting Get, best to show it:

```bash
set -e
USERNAME=$(choria kv watch CFG username --once --timeout 10s)
```

Here if there is a value at start time the value will be got and set in *USERNAME*, if there is not a value immediately
it will wait for up to 10 seconds for a value to be created.

If after 10 seconds nothing arrived a timeout will be reached and error raised

### Purging a bucket

All the values can be removed from a bucket and history will not be kept.  Note any active watchers will not be notified
that the purge has happened.

```nohighlight
$ choria kv purge CFG
? Really remove the CFG bucket Yes
$ choria kv info CFG
CFG Key-Value Store

     Bucket Name: CFG
   Values Stored: 0
         History: 5
             TTL: 0s
 Max Bucket Size: 102400
  Max Value Size: 10240
```

We can see after purge no values are stored.

### Removing a bucket

Finally, the bucket can be completely removed using `choria kv rm CFG`.

## Autonomous Agent Integration

Choria Autonomous Agents let us build small Kubernetes Operator like managed finite state machines. Since version *0.23.0*
they integrate with key-value stores and have a built-in data layer.

```yaml
watchers:
  - name: kv_watch
    type: kv
    interval: 10s
    state_match: [WAIT]
    success_transition: update_deployment
    properties:
      bucket: CFG
      key: username
      mode: poll
```

This will watch the bucket in a polling mode and if the *username* key update will transition to *update_deployment*.

Any *exec* watchers that will follow will have access to a new environment variable *WATCHER_DATA* that is a path to a JSON
file holding any data the machine manages.

Right now only a single key poll is supported, we'll add an entire bucket watch which will be more suitable for complex automations.
