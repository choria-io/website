+++
title = "Leader Elections"
weight = 40
+++

Choria sets up a Key-Value bucket that can be used by Go programs to perform Leader Elections.  Each key forms an Election
and the bucket as a whole has a TTL which represents the campaign interval.

The purpose is to allow applications like Choria Provisioner to do leader election without also having their own high-available
storage. This facility is available to any program on your network that needs a similar feature.

## Behavior

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
sets the TTL.  We strongly recomment defaults are kept.

## Inspecting from the CLI

The `choria` CLI can be used to see who is the current leader based on Choria Identity.

```nohighlight
$ choria kv get CHORIA_LEADER_ELECTION my_app
CHORIA_LEADER_ELECTION > my_app created @ 01 Nov 21 17:00 UTC

c1.example.net
```

A leadership stand-down can be forced by deleting this key:

```nohighlight
$ choria kv del CHORIA_LEADER_ELECTION my_app
? Really remove the my_app key from bucket CHORIA_LEADER_ELECTION Yes
```

After this, within a minute, the leader will stand-down and another worker will take over.

Listing the `keys` on the bucket will show you all elections:

```nohighlight
$ choria kv keys CHORIA_LEADER_ELECTION
my_app
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
