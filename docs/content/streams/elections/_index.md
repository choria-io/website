	+++
title = "Leader Elections"
weight = 40
+++

Choria sets up a Key-Value bucket that can be used by Go programs to perform Leader Elections.  Each key forms an Election
and the bucket as a whole has a TTL which represents the campaign interval.

The purpose is to allow applications like Choria Provisioner to do leader election without also having their own high-available
storage. This facility is available to any program on your network that needs a similar feature.

{{% notice tip %}}
Examples using the `choria election` command requires *Choria Server 0.26.1* or newer
{{% /notice %}}

## Behaviour

Once an Election is started, Campaigners will try to create a value in the KV Bucket with the option set to only succeed
if the key has no value in it currently.

Whoever has a successful create operation becomes the leader after a cool-down grace period and notes the value sequence.
The leader repeatedly updates the bucket with a new value. The update has the option set that ensures the last value in
the Key is the one that he put. As long as he is updating his own data he remains leader.

If ever the updates stop for a minute (or configured TTL) the data expires and another campaigner becomes the leader.

The design of this system is not overly aggressive on purpose, the implications is that leadership changes can take up to
a minute and new leaders can take a while to take up new work.  This is to ensure the system will scale to very large systems
at the cost of real time:

 * Campaigns are only done every minute (or more if a backoff is done)
 * The leader only maintains his leadership around once a minute
 * Once a new leader is elected a cool-off period of around 1 minute is entered to allow past leaders to stand down cleanly

## Configuration

When Choria Broker starts it will automatically create a KV Bucket called `CHORIA_LEADER_ELECTION` with a 1 minute TTL
by default. By default, the bucket will have Replicas set equalling the cluster size.

The `plugin.choria.network.stream.leader_election_replicas` setting sets the number of replicas and `plugin.choria.network.stream.leader_election_ttl`
sets the TTL.  We strongly recommend defaults are kept.

## Inspecting from the CLI

### Election Bucket State

Since version `0.26.1` the overall state of a bucket can be viewed:

```nohighlight
$ choria election info
Election bucket information for CHORIA_LEADER_ELECTION

       Created: 27 Jun 22 09:05 +0000
       Storage: File
  Maximum Time: 10s
      Replicas: 1
     Elections: 1

╭───────────────────────────────────────────────────────────────────────────╮
│                             Active Elections                              │
├────────────────┬──────────────────────────────────────────────────────────┤
│ Election       │ Leader                                                   │
├────────────────┼──────────────────────────────────────────────────────────┤
│ task_scheduler │ task-scheduler-asyncjobs-task-scheduler-6c96b69db5-zc8cr │
╰────────────────┴──────────────────────────────────────────────────────────╯
```

The `--bucket` argument is optional and will default to `CHORIA_LEADER_ELECTION`.
This command will also show warnings about general bucket health and so forth.

An optional argument will apply a regular expression over the keys in the `Active Election` table.

For earlier versions `choria` CLI can be used to see who is the current leader based on Choria Identity.

```nohighlight
$ choria kv get CHORIA_LEADER_ELECTION my_app
CHORIA_LEADER_ELECTION > my_app sequence 20 created @ 01 Nov 21 17:30 UTC

c1.example.net
```

### Evicting a leader

The current leader for an election can be evicted which will cause him to stand by and a new leader (possibly
the same one) can be elected.

Since version `0.26.1` the `election` utility can be used:

```nohighlight
$ choria election evict task_scheduler
? Evict the current leader from election task_scheduler in bucket CHORIA_LEADER_ELECTION Yes
Evicted the leader from election task_scheduler in bucket CHORIA_LEADER_ELECTION
```

In earlier versions a leadership stand-down can be forced by deleting this key:

```nohighlight
$ choria kv del CHORIA_LEADER_ELECTION task_scheduler
? Really remove the task_scheduler key from bucket CHORIA_LEADER_ELECTION Yes
```

After this, within a minute, the leader will stand-down and another worker will take over.

### Controlling the presence of a file

Often where cron is used for job scheduling a set of cron jobs need to run together but you also want
to support failing over to another server should the selected server fail.

There are many ways to accomplish this - best is perhaps to use a actual distributed job system - but cron
is nice and many tools like Puppet and Ansible support it and it's well understood.

Choria can maintain a file on a server using Elections that can signal to related cron jobs that they are
active on any given machine:

```nohighlight
$ choria election file /tmp/cron.host CRON
```

Here we campaign within an election `CRON` and on the leader create the file `/tmp/cron.host`.  This will 
be written frequently.

You cron jobs can now just check for the presence and age of this file to know if they should be active on a node.

```nohighlight
* * * * * root choria election file --check /tmp/cron.host && echo "hello world"
```

### Running a command under Election control

To run shell scripts and similar under Election control is difficult, we have a helper that can do that by starting
and signalling a process based on the election status.

```nohighlight
This feature is subject to change as we refine the model a bit, your feedback would be appreciated
```

The basic idea is that your shell script or program should trap `USR1`, `USR2` and `INT`. It will receive `USR1` when
an election is won, `USR2` when it is lost and when opting out of the signal mode and `INT` before `TERM` when the election
is lost.

```bash
#!/bin/bash

LEADER=0

function won() {
  echo "Election won"; LEADER=1
}
function lost() {
  echo "Election lost"; LEADER=0
}
function int() {
  echo "Got interrupt signal"; exit
}

trap won USR1
trap lost USR2
trap int INT

while true
do
  if [ $LEADER == 1 ]
  then
    echo "Doing work...."
  fi
  sleep 5
done
```

We can run this script in one of 2 modes. In both modes the script should assume it is not the leader at start and it
must always accept at least `USR1`.

First where it will receive `USR1`/`USR2` signals and run for a long time. 

```nohighlight
$ choria election run SHELL ./test.sh
Election won
Doing work....
Doing work....
WARN[0034] Sending SIGUSR2 to 1158614                    component=election
Election lost
```

In the second mode `USR2` will not be sent, the program wlll be terminated when leadership is lost:

```nohighlight
$ choria election run SHELL --terminate ./test.sh
Election won
Doing work....
Doing work....
WARN[0031] Sending SIGINT to 1158759                     component=election
WARN[0032] Sending SIGTERM to 1158759                    component=election
ERRO[0032] Execution failed with exit code: 1            component=election
$ echo $?
1
```

## Using from Go

Each group of related workers need to share a name - here we use `my_app` - and a number of optional callbacks are supported:

```go
fw, err := choria.New(choria.UserConfig())
if err != nil {
    panic(err)
}

ctx, cancel := context.WithCancel(context.Background())
defer cancel()

won := func() {
	// do winning tasks
}

lost := func() {
	// do losing tasks
}

elect, err := fw.NewElection(ctx, nil, "my_app", election.OnWon(won), election.OnLost(lost))
if err != nil {
	panic(err)
}

// election runs forever until context stops it
// you should already have background workers setup
// or do this in a go routine
err = elect.Start(ctx)
if err != nil {
	panic(err)
}
```

Here we create a `NewElection` that also takes care of creating a new connection, since we passed a nil `Connection`. It will campaign in the `my_app` election and will call `won` should it become leader and `lost` should it need to step down.

The campaign continues in the background until the context ends it.

On busy networks or those with many elections we suggest using the `election.BackOff(backoff.TwentySec)` election option, this will let stand-by nodes slow down their
campaigns.  It will have the effect that a failover will be slow but, on the other hand, if a leader has a short outages or there was a quick broker restart, the old
leader will most probably become leader again (he would be campaining much more frequently than other candidates).

A full example of an app starting 10 workers, in individual go routines to similate concurrent stand-by workers, and logging some lines from the leader can be seen below.

```go
package main

import (
	"context"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"

	"github.com/choria-io/go-choria/choria"
	"github.com/choria-io/go-choria/inter"
	election "github.com/choria-io/go-choria/providers/election/streams"
	"github.com/sirupsen/logrus"
)

type worker struct {
	i      int
	ctx    context.Context
	log    *logrus.Entry
	mu     sync.Mutex
	leader bool
	e      inter.Election
}

// callback when we become leader
func (w *worker) win() {
	w.mu.Lock()
	defer w.mu.Unlock()

	w.leader = true
	w.log.Infof("Became leader")
}

// callback on leader lost
func (w *worker) loose() {
	w.mu.Lock()
	defer w.mu.Unlock()

	w.leader = false
	w.log.Infof("Lost leadership")
}

func (w *worker) ifLeader(cb func()) bool {
	w.mu.Lock()
	leader := w.leader
	w.mu.Unlock()

	if leader {
		cb()
	}

	return leader
}

// every second logs a line if we are the leader, else noop
func (w *worker) doWork() {
	timer := time.NewTicker(time.Second)
	for {
		select {
		case <-timer.C:
			w.ifLeader(func() {
				w.log.Infof("Doing work as the leader")
			})

		case <-w.ctx.Done():
			return
		}
	}
}

func newWorker(ctx context.Context, wg *sync.WaitGroup, fw inter.Framework, i int) error {
	defer wg.Done()

	w := &worker{
		i:   i,
		ctx: ctx,
		log: fw.Logger("worker").WithField("count", i),
	}

	var err error

	// join the my_app election
	w.e, err = fw.NewElection(ctx, nil, "my_app", election.OnWon(w.win), election.OnLost(w.loose))
	if err != nil {
		return err
	}

	w.log.Infof("Worker starting")

	// schedule work should we become the leader
	go w.doWork()

	return w.e.Start(ctx)
}

func waitForInterrupt(ctx context.Context, cancel context.CancelFunc, log *logrus.Entry) {
	sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)

	for {
		select {
		case <-sigs:
			log.Infof("Terminating on interrupt")
			cancel()
			return
		case <-ctx.Done():
			return
		}
	}

}
func main() {
	fw, err := choria.New(choria.UserConfig())
	if err != nil {
		panic(err)
	}
	log := fw.Logger("manager")

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	go waitForInterrupt(ctx, cancel, log)

	wg := &sync.WaitGroup{}

	// start 10 workers
	for i := 1; i <= 10; i++ {
		go func(i int) {
			wg.Add(1)
			err = newWorker(ctx, wg, fw, i)
			if err != nil {
				log.Errorf("worker %d failed: %s", i, err)
			}
		}(i)
	}

	<-ctx.Done()
}
```
