---
layout: post
title: Know more about go concurrency bugs
date:  2019-07-10 14:33:06
categories: Golang
image: /assets/images/post.jpg
---

关于`go concurrency bugs`的学习记录，来源于[github项目](https://github.com/system-pclub/go-concurrency-bugs)

### blocking-bugs

#### kubernetes

```go
/*
Fix the deadlock by refactoring workers to acquire a read lock *after* work is
   popped from the queue. This allows writers to get locks while workers are idle,
   while preserving the worker pause semantics necessary to allow safe sync.
*/
diff --git a/pkg/controller/resourcequota/resource_quota_controller.go b/pkg/controller/resourcequota/resource_quota_controller.go
index b2ae6d1..e341e1c 100644
--- a/pkg/controller/resourcequota/resource_quota_controller.go
+++ b/pkg/controller/resourcequota/resource_quota_controller.go

@@ -237,15 +237,13 @@ func (rq *ResourceQuotaController) addQuota(obj interface{}) {
 // worker runs a worker thread that just dequeues items, processes them, and marks them done.
 func (rq *ResourceQuotaController) worker(queue workqueue.RateLimitingInterface) func() {
 	workFunc := func() bool {
-
-		rq.workerLock.RLock()
-		defer rq.workerLock.RUnlock()
-
 		key, quit := queue.Get()
 		if quit {
 			return true
 		}
 		defer queue.Done(key)
+		rq.workerLock.RLock() // 在 queue.Get()之后进行RLock
+		defer rq.workerLock.RUnlock()
 		err := rq.syncHandler(key.(string))
 		if err == nil {

diff --git a/pkg/watch/mux.go b/pkg/watch/mux.go
index 0c87111..2d40014 100644
--- a/pkg/watch/mux.go
+++ b/pkg/watch/mux.go
@@ -56,9 +56,10 @@ func (m *Mux) Watch() Interface {
 	id := m.nextWatcher
 	m.nextWatcher++
 	w := &muxWatcher{
-		result: make(chan Event),
-		id:     id,
-		m:      m,
+		result:  make(chan Event),
+		stopped: make(chan struct{}),
+		id:      id,
+		m:       m,
 	}
 	m.watchers[id] = w
 	return w
@@ -119,15 +120,20 @@ func (m *Mux) distribute(event Event) {
 	m.lock.Lock()
 	defer m.lock.Unlock()
 	for _, w := range m.watchers {
-		w.result <- event  
+		select {  // 使用select+stopped避免event的阻塞,这是通用的范式
+		case w.result <- event:
+		case <-w.stopped:
+		}
 	}
 }

 // muxWatcher handles a single watcher of a mux
 type muxWatcher struct {
-	result chan Event
-	id     int64
-	m      *Mux
+	result  chan Event
+	stopped chan struct{}
+	stop    sync.Once
+	id      int64
+	m       *Mux
 }

 // ResultChan returns a channel to use for waiting on events.
@@ -137,5 +143,8 @@ func (mw *muxWatcher) ResultChan() <-chan Event {

 // Stop stops watching and removes mw from its list.
 func (mw *muxWatcher) Stop() {
-	mw.m.stopWatching(mw.id)
+	mw.stop.Do(func() {
+		close(mw.stopped)
+		mw.m.stopWatching(mw.id)
+	})
 }


 diff --git a/federation/pkg/federation-controller/util/federated_informer.go b/federation/pkg/federation-controller/util/federated_informer.go
 index 24de649..19f7b8b 100644
 --- a/federation/pkg/federation-controller/util/federated_informer.go
 +++ b/federation/pkg/federation-controller/util/federated_informer.go
 @@ -344,8 +344,6 @@ func (f *federatedInformerImpl) getReadyClusterUnlocked(name string) (*federatio

  // Synced returns true if the view is synced (for the first time)
  func (f *federatedInformerImpl) ClustersSynced() bool {
 -	f.Lock()
 -	defer f.Unlock()
  	return f.clusterInformer.controller.HasSynced()
  }

 @@ -452,18 +450,31 @@ func (fs *federatedStoreImpl) GetKeyFor(item interface{}) string {
  // Checks whether stores for all clusters form the lists (and only these) are there and
  // are synced.
  func (fs *federatedStoreImpl) ClustersSynced(clusters []*federation_api.Cluster) bool {
 -	fs.federatedInformer.Lock()
 -	defer fs.federatedInformer.Unlock()

 -	if len(fs.federatedInformer.targetInformers) != len(clusters) {
 +	// Get the list of informers to check under a lock and check it outside.
 +	okSoFar, informersToCheck := func() (bool, []informer) {
 +		fs.federatedInformer.Lock()
 +		defer fs.federatedInformer.Unlock()
 +
 +		if len(fs.federatedInformer.targetInformers) != len(clusters) {
 +			return false, []informer{}
 +		}
 +		informersToCheck := make([]informer, 0, len(clusters))
 +		for _, cluster := range clusters {
 +			if targetInformer, found := fs.federatedInformer.targetInformers[cluster.Name]; found {
 +				informersToCheck = append(informersToCheck, targetInformer)
 +			} else {
 +				return false, []informer{}
 +			}
 +		}
 +		return true, informersToCheck
 +	}()
 +
 +	if !okSoFar {
  		return false
  	}
 -	for _, cluster := range clusters {
 -		if targetInformer, found := fs.federatedInformer.targetInformers[cluster.Name]; found {
 -			if !targetInformer.controller.HasSynced() {
 -				return false
 -			}
 -		} else {
 +	for _, informerToCheck := range informersToCheck {
 +		if !informerToCheck.controller.HasSynced() {
  			return false
  		}
    }

    diff --git a/Godeps/_workspace/src/github.com/docker/spdystream/connection.go b/Godeps/_workspace/src/github.com/docker/spdystream/connection.go
    index 846b934..6e623c4 100644
    --- a/Godeps/_workspace/src/github.com/docker/spdystream/connection.go
    +++ b/Godeps/_workspace/src/github.com/docker/spdystream/connection.go
    @@ -89,10 +89,25 @@ Loop:
     			if timer != nil {
     				timer.Stop()
     			}
    +
    +			// Start a goroutine to drain resetChan. This is needed because we've seen
    +			// some unit tests with large numbers of goroutines get into a situation
    +			// where resetChan fills up, at least 1 call to Write() is still trying to
    +			// send to resetChan, the connection gets closed, and this case statement
    +			// attempts to grab the write lock that Write() already has, causing a
    +			// deadlock.
    +			//
    +			// See https://github.com/docker/spdystream/issues/49 for more details.
    +			go func() {
    +				for _ = range resetChan { // drain the resetChan
    +				}
    +			}()
    +
     			i.writeLock.Lock()
     			close(resetChan)
     			i.resetChan = nil
     			i.writeLock.Unlock()
    +
     			break Loop
     		}
      }


Previously, we had no way to stop resync goroutine when ListAndWatch
returned.  goroutine leaked every time ListAndWatch returned, for
example, with error.  This commit adds another channel to signal that
resync goroutine should exit when ListAndWatch returns.

diff --git a/pkg/client/cache/reflector.go b/pkg/client/cache/reflector.go
index 8c8aee3..785cab2 100644
--- a/pkg/client/cache/reflector.go
+++ b/pkg/client/cache/reflector.go
@@ -259,12 +259,16 @@ func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
 	r.setLastSyncResourceVersion(resourceVersion)

 	resyncerrc := make(chan error, 1)
+	cancelCh := make(chan struct{})
+	defer close(cancelCh) // 最后close 能够取消下面的goroutine
 	go func() {
 		for {
 			select {
 			case <-resyncCh:
 			case <-stopCh:
 				return
+			case <-cancelCh:
+				return
 			}
 			glog.V(4).Infof("%s: forcing resync", r.name)
 			if err := r.store.Resync(); err != nil {

func main() {
	fmt.Println("Hello, playground")
	// fmt.Println(UniqRands(13,13))
	rchan := make(chan int)
	cancelCh := make(chan struct{})

	go func() { // goroutine1
		for {
			select {
			case <-rchan:
			case <-cancelCh:
				fmt.Println("=======")
				return
			}
		}
	}()

+ Cc(cancelCh) // 保证goroutine1不会因为rchan发生阻塞而永不结束而阻塞

}

func Cc(cancelCh chan struct{}) {
	defer close(cancelCh)
	fmt.Println("Close cancelCh")
}


avoid dobule RLock() in cpumanager

diff --git a/pkg/kubelet/cm/cpumanager/state/state_mem.go b/pkg/kubelet/cm/cpumanager/state/state_mem.go
index 797cdb1..816c898 100644
--- a/pkg/kubelet/cm/cpumanager/state/state_mem.go
+++ b/pkg/kubelet/cm/cpumanager/state/state_mem.go
@@ -56,9 +56,6 @@ func (s *stateMemory) GetDefaultCPUSet() cpuset.CPUSet {
 }

 func (s *stateMemory) GetCPUSetOrDefault(containerID string) cpuset.CPUSet {
-	s.RLock()
-	defer s.RUnlock()
-
 	if res, ok := s.GetCPUSet(containerID); ok {  // GetCPUSet方法中已经有锁了，外层不应该再用锁，避免嵌套加锁
 		return res
 	}
```

#### etcd

```go
diff --git a/contrib/raftexample/raft.go b/contrib/raftexample/raft.go
index 736e988..1ac5682 100644
--- a/contrib/raftexample/raft.go
+++ b/contrib/raftexample/raft.go
@@ -41,10 +41,11 @@ type raftNode struct {
 	commitC     chan *string             // entries committed to log (k,v)
 	errorC      chan error               // errors from raft session

-	id     int      // client ID for raft session
-	peers  []string // raft peer URLs
-	join   bool     // node is joining an existing cluster
-	waldir string   // path to WAL directory
+	id        int      // client ID for raft session
+	peers     []string // raft peer URLs
+	join      bool     // node is joining an existing cluster
+	waldir    string   // path to WAL directory
+	lastIndex uint64   // index of log at start

 	// raft backing for the commit/error channel
 	node        raft.Node
@@ -90,8 +91,8 @@ func (rc *raftNode) publishEntries(ents []raftpb.Entry) bool {
 		switch ents[i].Type {
 		case raftpb.EntryNormal:
 			if len(ents[i].Data) == 0 {
-				// ignore conf changes and empty messages
-				continue
+				// ignore empty messages
+				break
 			}
 			s := string(ents[i].Data)
 			select {
@@ -103,7 +104,6 @@ func (rc *raftNode) publishEntries(ents []raftpb.Entry) bool {
 		case raftpb.EntryConfChange:
 			var cc raftpb.ConfChange
 			cc.Unmarshal(ents[i].Data)
-
 			rc.node.ApplyConfChange(cc)
 			switch cc.Type {
 			case raftpb.ConfChangeAddNode:
@@ -118,6 +118,15 @@ func (rc *raftNode) publishEntries(ents []raftpb.Entry) bool {
 				rc.transport.RemovePeer(types.ID(cc.NodeID))
 			}
 		}
+
+		// special nil commit to signal replay has finished
+		if ents[i].Index == rc.lastIndex {
+			select {
+			case rc.commitC <- nil:
+			case <-rc.stopc:
+				return false
+			}
+		}
 	}
 	return true
 }
@@ -144,19 +153,22 @@ func (rc *raftNode) openWAL() *wal.WAL {
 	return w
 }

-// replayWAL replays WAL entries into the raft instance and the commit
-// channel and returns an appendable WAL.
+// replayWAL replays WAL entries into the raft instance.
 func (rc *raftNode) replayWAL() *wal.WAL {
 	w := rc.openWAL()
-	_, _, ents, err := w.ReadAll()
+	_, st, ents, err := w.ReadAll()
 	if err != nil {
 		log.Fatalf("raftexample: failed to read WAL (%v)", err)
 	}
 	// append to storage so raft starts at the right place in log
 	rc.raftStorage.Append(ents)
-	rc.publishEntries(ents)
-	// send nil value so client knows commit channel is current
-	rc.commitC <- nil
+	// send nil once lastIndex is published so client knows commit channel is current
+	if len(ents) > 0 {
+		rc.lastIndex = ents[len(ents)-1].Index
+	} else {
+		rc.commitC <- nil
+	}
+	rc.raftStorage.SetHardState(st)
 	return w
 }

 diff --git a/clientv3/concurrency/election.go b/clientv3/concurrency/election.go
 index 1259e0f..89b5208 100644
 --- a/clientv3/concurrency/election.go
 +++ b/clientv3/concurrency/election.go
 @@ -16,6 +16,7 @@ package concurrency

  import (
  	"errors"
 +	"fmt"

  	v3 "github.com/coreos/etcd/clientv3"
  	"github.com/coreos/etcd/mvcc/mvccpb"
 @@ -50,22 +51,39 @@ func (e *Election) Campaign(ctx context.Context, val string) error {
  		return serr
  	}

 -	k, rev, err := NewUniqueKV(ctx, e.client, e.keyPrefix, val, v3.WithLease(s.Lease()))
 -	if err == nil {
 -		err = waitDeletes(ctx, e.client, e.keyPrefix, v3.WithPrefix(), v3.WithRev(rev-1))
 +	k := fmt.Sprintf("%s/%x", e.keyPrefix, s.Lease())
 +	txn := e.client.Txn(ctx).If(v3.Compare(v3.CreateRevision(k), "=", 0))
 +	txn = txn.Then(v3.OpPut(k, val, v3.WithLease(s.Lease())))
 +	txn = txn.Else(v3.OpGet(k))
 +	resp, err := txn.Commit()
 +	if err != nil {
 +		return err
 +	}
 +
 +	e.leaderKey, e.leaderRev, e.leaderSession = k, resp.Header.Revision, s
 +	if !resp.Succeeded {
 +		kv := resp.Responses[0].GetResponseRange().Kvs[0]
 +		e.leaderRev = kv.CreateRevision
 +		if string(kv.Value) != val {
 +			if err = e.Proclaim(ctx, val); err != nil {
 +				e.Resign(ctx)
 +				return err
 +			}
 +		}
  	}

 +	err = waitDeletes(ctx, e.client, e.keyPrefix, v3.WithPrefix(), v3.WithRev(e.leaderRev-1))
  	if err != nil {
  		// clean up in case of context cancel
  		select {
  		case <-ctx.Done():
 -			e.client.Delete(e.client.Ctx(), k)
 +			e.Resign(e.client.Ctx())
  		default:
 +			e.leaderSession = nil
  		}
  		return err
  	}

 -	e.leaderKey, e.leaderRev, e.leaderSession = k, rev, s
  	return nil
  }

 diff --git a/clientv3/concurrency/key.go b/clientv3/concurrency/key.go
 index 65e2b89..74d495d 100644
 --- a/clientv3/concurrency/key.go
 +++ b/clientv3/concurrency/key.go
 @@ -17,34 +17,12 @@ package concurrency
  import (
  	"fmt"
  	"math"
 -	"time"

  	v3 "github.com/coreos/etcd/clientv3"
  	"github.com/coreos/etcd/mvcc/mvccpb"
  	"golang.org/x/net/context"
  )

 -// NewUniqueKey creates a new key from a given prefix.
 -func NewUniqueKey(ctx context.Context, kv v3.KV, pfx string, opts ...v3.OpOption) (string, int64, error) {
 -	return NewUniqueKV(ctx, kv, pfx, "", opts...)
 -}
 -
 -func NewUniqueKV(ctx context.Context, kv v3.KV, pfx, val string, opts ...v3.OpOption) (string, int64, error) {
 -	for {
 -		newKey := fmt.Sprintf("%s/%v", pfx, time.Now().UnixNano())
 -		put := v3.OpPut(newKey, val, opts...)
 -		cmp := v3.Compare(v3.ModRevision(newKey), "=", 0)
 -		resp, err := kv.Txn(ctx).If(cmp).Then(put).Commit()
 -		if err != nil {
 -			return "", 0, err
 -		}
 -		if !resp.Succeeded {
 -			continue
 -		}
 -		return newKey, resp.Header.Revision, nil
 -	}
 -}
 -
  func waitDelete(ctx context.Context, client *v3.Client, key string, rev int64) error {
  	cctx, cancel := context.WithCancel(ctx)
  	defer cancel()
 diff --git a/integration/v3_election_test.go b/integration/v3_election_test.go
 index d355986..dcadd78 100644
 --- a/integration/v3_election_test.go
 +++ b/integration/v3_election_test.go
 @@ -146,3 +146,26 @@ func TestElectionFailover(t *testing.T) {
  	// leader must ack election (otherwise, Campaign may see closed conn)
  	<-electedc
  }
 +
 +// TestElectionSessionRelock ensures that campaigning twice on the same election
 +// with the same lock will Proclaim instead of deadlocking.
 +func TestElectionSessionRecampaign(t *testing.T) {
 +	clus := NewClusterV3(t, &ClusterConfig{Size: 1})
 +	defer clus.Terminate(t)
 +	cli := clus.RandClient()
 +
 +	e := concurrency.NewElection(cli, "test-elect")
 +	if err := e.Campaign(context.TODO(), "abc"); err != nil {
 +		t.Fatal(err)
 +	}
 +	e2 := concurrency.NewElection(cli, "test-elect")
 +	if err := e2.Campaign(context.TODO(), "def"); err != nil {
 +		t.Fatal(err)
 +	}
 +
 +	ctx, cancel := context.WithCancel(context.TODO())
 +	defer cancel()
 +	if resp := <-e.Observe(ctx); len(resp.Kvs) == 0 || string(resp.Kvs[0].Value) != "def" {
 +		t.Fatalf("expected value=%q, got response %v", "def", resp)
 +	}
 +}

 diff --git a/auth/simple_token.go b/auth/simple_token.go
 index 5b608af..ff48c51 100644
 --- a/auth/simple_token.go
 +++ b/auth/simple_token.go
 @@ -32,27 +32,26 @@ import (
  const (
  	letters                  = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
  	defaultSimpleTokenLength = 16
 +)
 +
 +// var for testing purposes
 +var (
  	simpleTokenTTL           = 5 * time.Minute
  	simpleTokenTTLResolution = 1 * time.Second
  )

  type simpleTokenTTLKeeper struct {
 -	tokens              map[string]time.Time
 -	addSimpleTokenCh    chan string
 -	resetSimpleTokenCh  chan string
 -	deleteSimpleTokenCh chan string
 -	stopCh              chan chan struct{}
 -	deleteTokenFunc     func(string)
 +	tokensMu        sync.Mutex
 +	tokens          map[string]time.Time
 +	stopCh          chan chan struct{}
 +	deleteTokenFunc func(string)
  }

  func NewSimpleTokenTTLKeeper(deletefunc func(string)) *simpleTokenTTLKeeper {
  	stk := &simpleTokenTTLKeeper{
 -		tokens:              make(map[string]time.Time),
 -		addSimpleTokenCh:    make(chan string, 1),
 -		resetSimpleTokenCh:  make(chan string, 1),
 -		deleteSimpleTokenCh: make(chan string, 1),
 -		stopCh:              make(chan chan struct{}),
 -		deleteTokenFunc:     deletefunc,
 +		tokens:          make(map[string]time.Time),
 +		stopCh:          make(chan chan struct{}),
 +		deleteTokenFunc: deletefunc,
  	}
  	go stk.run()
  	return stk
 @@ -66,37 +65,34 @@ func (tm *simpleTokenTTLKeeper) stop() {
  }

  func (tm *simpleTokenTTLKeeper) addSimpleToken(token string) {
 -	tm.addSimpleTokenCh <- token
 +	tm.tokens[token] = time.Now().Add(simpleTokenTTL)
  }

  func (tm *simpleTokenTTLKeeper) resetSimpleToken(token string) {
 -	tm.resetSimpleTokenCh <- token
 +	if _, ok := tm.tokens[token]; ok {
 +		tm.tokens[token] = time.Now().Add(simpleTokenTTL)
 +	}
  }

  func (tm *simpleTokenTTLKeeper) deleteSimpleToken(token string) {
 -	tm.deleteSimpleTokenCh <- token
 +	delete(tm.tokens, token)
  }
 +
  func (tm *simpleTokenTTLKeeper) run() {
  	tokenTicker := time.NewTicker(simpleTokenTTLResolution)
  	defer tokenTicker.Stop()
  	for {
  		select {
 -		case t := <-tm.addSimpleTokenCh:
 -			tm.tokens[t] = time.Now().Add(simpleTokenTTL)
 -		case t := <-tm.resetSimpleTokenCh:
 -			if _, ok := tm.tokens[t]; ok {
 -				tm.tokens[t] = time.Now().Add(simpleTokenTTL)
 -			}
 -		case t := <-tm.deleteSimpleTokenCh:
 -			delete(tm.tokens, t)
  		case <-tokenTicker.C:
  			nowtime := time.Now()
 +			tm.tokensMu.Lock()
  			for t, tokenendtime := range tm.tokens {
  				if nowtime.After(tokenendtime) {
  					tm.deleteTokenFunc(t)
  					delete(tm.tokens, t)
  				}
  			}
 +			tm.tokensMu.Unlock()
  		case waitCh := <-tm.stopCh:
  			tm.tokens = make(map[string]time.Time)
  			waitCh <- struct{}{}
 @@ -108,7 +104,7 @@ func (tm *simpleTokenTTLKeeper) run() {
  type tokenSimple struct {
  	indexWaiter       func(uint64) <-chan struct{}
  	simpleTokenKeeper *simpleTokenTTLKeeper
 -	simpleTokensMu    sync.RWMutex
 +	simpleTokensMu    sync.Mutex
  	simpleTokens      map[string]string // token -> username
  }

 @@ -128,6 +124,7 @@ func (t *tokenSimple) genTokenPrefix() (string, error) {
  }

  func (t *tokenSimple) assignSimpleTokenToUser(username, token string) {
 +	t.simpleTokenKeeper.tokensMu.Lock()
  	t.simpleTokensMu.Lock()

  	_, ok := t.simpleTokens[token]
 @@ -138,18 +135,23 @@ func (t *tokenSimple) assignSimpleTokenToUser(username, token string) {
  	t.simpleTokens[token] = username
  	t.simpleTokenKeeper.addSimpleToken(token)
  	t.simpleTokensMu.Unlock()
 +	t.simpleTokenKeeper.tokensMu.Unlock()
  }

  func (t *tokenSimple) invalidateUser(username string) {
 +	if t.simpleTokenKeeper == nil {
 +		return
 +	}
 +	t.simpleTokenKeeper.tokensMu.Lock()
  	t.simpleTokensMu.Lock()
 -	defer t.simpleTokensMu.Unlock()
 -
  	for token, name := range t.simpleTokens {
  		if strings.Compare(name, username) == 0 {
  			delete(t.simpleTokens, token)
  			t.simpleTokenKeeper.deleteSimpleToken(token)
  		}
  	}
 +	t.simpleTokensMu.Unlock()
 +	t.simpleTokenKeeper.tokensMu.Unlock()
  }

  func newDeleterFunc(t *tokenSimple) func(string) {
 @@ -172,7 +174,6 @@ func (t *tokenSimple) disable() {
  		t.simpleTokenKeeper.stop()
  		t.simpleTokenKeeper = nil
  	}
 -
  	t.simpleTokensMu.Lock()
  	t.simpleTokens = make(map[string]string) // invalidate all tokens
  	t.simpleTokensMu.Unlock()
 @@ -182,14 +183,14 @@ func (t *tokenSimple) info(ctx context.Context, token string, revision uint64) (
  	if !t.isValidSimpleToken(ctx, token) {
  		return nil, false
  	}
 -
 -	t.simpleTokensMu.RLock()
 -	defer t.simpleTokensMu.RUnlock()
 +	t.simpleTokenKeeper.tokensMu.Lock()
 +	t.simpleTokensMu.Lock()
  	username, ok := t.simpleTokens[token]
  	if ok {
  		t.simpleTokenKeeper.resetSimpleToken(token)
  	}
 -
 +	t.simpleTokensMu.Unlock()
 +	t.simpleTokenKeeper.tokensMu.Unlock()
  	return &AuthInfo{Username: username, Revision: revision}, ok
  }

diff --git a/raft/node.go b/raft/node.go
index 5fce584..c8410fd 100644
--- a/raft/node.go
+++ b/raft/node.go
@@ -462,8 +462,12 @@ func (n *node) ApplyConfChange(cc pb.ConfChange) *pb.ConfState {

 func (n *node) Status() Status {
 	c := make(chan Status)
-	n.status <- c
-	return <-c
+	select {
+	case n.status <- c:
+		return <-c // 会阻塞在这
+	case <-n.done:
+		return Status{}
+	}
 }

 diff --git a/proxy/grpcproxy/watch_broadcasts.go b/proxy/grpcproxy/watch_broadcasts.go
 index fc18b74..3ca6fa2 100644
 --- a/proxy/grpcproxy/watch_broadcasts.go
 +++ b/proxy/grpcproxy/watch_broadcasts.go
 @@ -116,13 +116,12 @@ func (wbs *watchBroadcasts) empty() bool { return len(wbs.bcasts) == 0 }

  func (wbs *watchBroadcasts) stop() {
  	wbs.mu.Lock()
 -	defer wbs.mu.Unlock()
 -
  	for wb := range wbs.bcasts {
  		wb.stop()
  	}
  	wbs.bcasts = nil
  	close(wbs.updatec)
 +	wbs.mu.Unlock()
  	<-wbs.donec // 按照原有方式，如果这里一直没有done一直阻塞，则锁就不会释放而deadlock
  }

  var b []byte
  -	errc := make(chan error)
  +	// buffer errc channel so that errc don't block inside the go routinue
  +	errc := make(chan error, 2)

  func (b *simpleBalancer) Close() error {
   	b.mu.Lock()
  -	defer b.mu.Unlock()
   	// In case gRPC calls close twice. TODO: remove the checking
   	// when we are sure that gRPC wont call close twice.
   	if b.closed {
  +		b.mu.Unlock()
  +		<-b.donec // 在 阻塞chan前、return 前 先解锁
   		return nil
   	}
   	b.closed = true
  -	close(b.notifyCh)
  +	close(b.stopc)
   	b.pinAddr = ""

   	// In the case of following scenario:
  @@ -236,6 +292,13 @@ func (b *simpleBalancer) Close() error {
   		// terminate all waiting Get()s
   		close(b.upc)
   	}
  +
  +	b.mu.Unlock()
  +
  +	// wait for updateNotifyLoop to finish
  +	<-b.donec
  +	close(b.notifyCh)
  +
   	return nil
   }

   diff --git a/contrib/raftexample/raft.go b/contrib/raftexample/raft.go
   index 555f2f5..753cc2a 100644
   --- a/contrib/raftexample/raft.go
   +++ b/contrib/raftexample/raft.go
   @@ -184,21 +184,23 @@ func (rc *raftNode) serveChannels() {
    	ticker := time.NewTicker(100 * time.Millisecond)
    	defer ticker.Stop()

   -	// event loop on client proposals and raft updates
   +	// send proposals over raft
   +	stopc := make(chan struct{}, 1)
   +	go func() {
   +		for prop := range rc.proposeC {
   +			// blocks until accepted by raft state machine
   +			rc.node.Propose(context.TODO(), []byte(prop))
   +		}
   +		// client closed channel; shutdown raft if not already
   +		stopc <- struct{}{}
   +	}()
   +
   +	// event loop on raft state machine updates
    	for {
    		select {
    		case <-ticker.C:
    			rc.node.Tick()

   -		// send proposals over raft
   -		case prop, ok := <-rc.proposeC:
   -			if !ok {
   -				// client closed channel; shut down
   -				rc.stop()
   -				return
   -			}
   -			rc.node.Propose(context.TODO(), []byte(prop))
   -
    		// store raft entries to wal, then publish over commit channel
    		case rd := <-rc.node.Ready():
    			rc.wal.Save(rd.HardState, rd.Entries)
   @@ -210,6 +212,10 @@ func (rc *raftNode) serveChannels() {
    		case err := <-rc.transport.ErrorC:
    			rc.writeError(err)
    			return
   +
   +		case <-stopc:
   +			rc.stop()
   +			return
    		}
    	}
    }

    diff --git a/client/client.go b/client/client.go
    index da86a0b..ece4cc0 100644
    --- a/client/client.go
    +++ b/client/client.go
    @@ -378,9 +378,12 @@ func (c *simpleHTTPClient) Do(ctx context.Context, act httpAction) (*http.Respon
     		return nil, nil, err
     	}

    -	hctx, hcancel := context.WithCancel(ctx)
    +	var hctx context.Context
    +	var hcancel context.CancelFunc
     	if c.headerTimeout > 0 {
     		hctx, hcancel = context.WithTimeout(ctx, c.headerTimeout)
    +	} else {
    +		hctx, hcancel = context.WithCancel(ctx)
     	}
    defer hcancel()

    diff --git a/clientv3/client.go b/clientv3/client.go
    index 4ec061d..99ea726 100644
    --- a/clientv3/client.go
    +++ b/clientv3/client.go
    @@ -87,12 +87,13 @@ func NewFromURL(url string) (*Client, error) {
     // Close shuts down the client's etcd connections.
     func (c *Client) Close() error {
     	c.mu.Lock()
    -	defer c.mu.Unlock()
     	if c.cancel == nil {
    +		c.mu.Unlock()
     		return nil
     	}
     	c.cancel()
     	c.cancel = nil
    +	c.mu.Unlock()
     	c.Watcher.Close()
     	c.Lease.Close()
     	return c.conn.Close()
    @@ -126,14 +127,22 @@ func (c *Client) Dial(endpoint string) (*grpc.ClientConn, error) {
     	} else {
     		opts = append(opts, grpc.WithInsecure())
     	}
    +
    +	proto := "tcp"
     	if url, uerr := url.Parse(endpoint); uerr == nil && url.Scheme == "unix" {
    -		f := func(a string, t time.Duration) (net.Conn, error) {
    -			return net.DialTimeout("unix", a, t)
    -		}
    +		proto = "unix"
     		// strip unix:// prefix so certs work
     		endpoint = url.Host
    -		opts = append(opts, grpc.WithDialer(f))
     	}
    +	f := func(a string, t time.Duration) (net.Conn, error) {
    +		select {
    +		case <-c.ctx.Done():
    +			return nil, c.ctx.Err()
    +		default:
    +		}
    +		return net.DialTimeout(proto, a, t)
    +	}
    +	opts = append(opts, grpc.WithDialer(f))

     	conn, err := grpc.Dial(endpoint, opts...)
     	if err != nil {
    @@ -156,11 +165,11 @@ func newClient(cfg *Config) (*Client, error) {
     		creds = &c
     	}
     	// use a temporary skeleton client to bootstrap first connection
    -	conn, err := cfg.RetryDialer(&Client{cfg: *cfg, creds: creds})
    +	ctx, cancel := context.WithCancel(context.TODO())
    +	conn, err := cfg.RetryDialer(&Client{cfg: *cfg, creds: creds, ctx: ctx})
     	if err != nil {
     		return nil, err
     	}
    -	ctx, cancel := context.WithCancel(context.TODO())
     	client := &Client{
     		conn:   conn,
     		cfg:    *cfg,
    @@ -198,6 +207,13 @@ func (c *Client) retryConnection(oldConn *grpc.ClientConn, err error) (*grpc.Cli
     		// conn has already been updated
     		return c.conn, nil
     	}
    +
    +	oldConn.Close()
    +	if st, _ := oldConn.State(); st != grpc.Shutdown {
    +		// wait for shutdown so grpc doesn't leak sleeping goroutines
    +		oldConn.WaitForStateChange(c.ctx, st)
    +	}
    +
     	conn, dialErr := c.cfg.RetryDialer(c)
     	if dialErr != nil {
     		c.errors = append(c.errors, dialErr)
```

#### grpc-go

```go
diff --git a/balancer.go b/balancer.go
index 2acc882..57c20c6 100644
--- a/balancer.go
+++ b/balancer.go
@@ -201,6 +201,10 @@ func (rr *roundRobin) watchAddrUpdates() error {
 	if rr.done {
 		return ErrClientConnClosing
 	}
+	select {
+	case <-rr.addrCh:
+	default:
+	}
 	rr.addrCh <- open
 	return nil
 }

 diff --git a/stream.go b/stream.go
 index 0ee572c..537d4b3 100644
 --- a/stream.go
 +++ b/stream.go
 @@ -133,8 +133,12 @@ func NewClientStream(ctx context.Context, desc *StreamDesc, cc *ClientConn, meth
  	// Listen on ctx.Done() to detect cancellation when there is no pending
  	// I/O operations on this stream.
  	go func() {
 -		<-s.Context().Done()
 -		cs.closeTransportStream(transport.ContextErr(s.Context().Err()))
 +		select {
 +		case <-t.Error():
 +			// Incur transport error, simply exit.
 +		case <-s.Context().Done():
 +			cs.closeTransportStream(transport.ContextErr(s.Context().Err()))
 +		}
  	}()
  	return cs, nil
  }

  diff --git a/benchmark/worker/benchmark_client.go b/benchmark/worker/benchmark_client.go
  index 7d14007..2ee5dbb 100644
  --- a/benchmark/worker/benchmark_client.go
  +++ b/benchmark/worker/benchmark_client.go
  @@ -200,8 +200,8 @@ func (bc *benchmarkClient) doCloseLoopUnary(conns []*grpc.ClientConn, rpcCountPe
   		for j := 0; j < rpcCountPerConn; j++ {
   			go func(client testpb.BenchmarkServiceClient) {
   				defer wg.Done()
  +				done := make(chan bool)
   				for {
  -					done := make(chan bool)
   					go func() {
   						start := time.Now()
   						if err := benchmark.DoUnaryCall(client, reqSize, respSize); err != nil {
  @@ -212,7 +212,10 @@ func (bc *benchmarkClient) doCloseLoopUnary(conns []*grpc.ClientConn, rpcCountPe
   						bc.mu.Lock()
   						bc.histogram.Add(int64(elapse / time.Nanosecond))
   						bc.mu.Unlock()
  -						done <- true
  +						select {
  +						case <-bc.stop:
  +						case done <- true:
  +						}
   					}()
   					select {
   					case <-bc.stop:
  @@ -259,8 +262,8 @@ func (bc *benchmarkClient) doCloseLoopStreaming(conns []*grpc.ClientConn, rpcCou
   		for j := 0; j < rpcCountPerConn; j++ {
   			go func(stream testpb.BenchmarkService_StreamingCallClient) {
   				defer wg.Done()
  +				done := make(chan bool)
   				for {
  -					done := make(chan bool) // 在for中make chan 会产生大量的chan
   					go func() {
   						start := time.Now()
   						if err := doRPC(stream, reqSize, respSize); err != nil {
  @@ -271,7 +274,10 @@ func (bc *benchmarkClient) doCloseLoopStreaming(conns []*grpc.ClientConn, rpcCou
   						bc.mu.Lock()
   						bc.histogram.Add(int64(elapse / time.Nanosecond))
   						bc.mu.Unlock()
  -						done <- true
  +						select {
  +						case <-bc.stop:
  +						case done <- true:
  +						}
   					}()

select 是非确定性的，stopCh 和 ticker 同时发生时，不一定会执行 stopChan 的分支，正确做法是先检查一次 stopCh

select {
case <-bc.stop:
ticker := time.NewTicker()
for {
    select{
        case <- stopCh:
            return
        default:
    }
    f()
    select {
        case <- stopCh:
            return
        case <- ticker:
    }
}

var group sync.WaitGroup
group.Add(len(pm.plugins))
for_, p := range pm.plugins {
    go func(p *plugin) {
        defer group.Done()
    }
    group.Wait() // 阻塞
}
// 应该在这里group.Wait()

func goroutine1() {
    m.Lock()
    ch <- request // 阻塞
    m.Unlock()
}

func goroutine2() {
    for{
        m.Lock()    // 阻塞
        m.Unlock()
        request <- ch
    }
}


diff --git a/daemon/logger/jsonfilelog/read.go b/daemon/logger/jsonfilelog/read.go
index 0ac0cb3..8794e89 100644
--- a/daemon/logger/jsonfilelog/read.go
+++ b/daemon/logger/jsonfilelog/read.go
@@ -9,6 +9,7 @@ import (
 	"os"
 	"time"

+	"golang.org/x/net/context"
 	"gopkg.in/fsnotify.v1"

 	"github.com/Sirupsen/logrus"
@@ -172,9 +173,22 @@ func followLogs(f *os.File, logWatcher *logger.LogWatcher, notifyRotate chan int
 	}
 	defer func() {
 		f.Close()
+		fileWatcher.Remove(name)
 		fileWatcher.Close()
 	}()

+	ctx, cancel := context.WithCancel(context.Background())
+	defer cancel()
+	go func() {
+		select {
+		case <-logWatcher.WatchClose():
+			fileWatcher.Remove(name)
+			cancel()
+		case <-ctx.Done():
+			return
+		}
+	}()
+
 	var retries int
 	handleRotate := func() error {
 		f.Close()
@@ -209,8 +223,7 @@ func followLogs(f *os.File, logWatcher *logger.LogWatcher, notifyRotate chan int
 			case fsnotify.Rename, fsnotify.Remove:
 				select {
 				case <-notifyRotate:
-				case <-logWatcher.WatchClose():
-					fileWatcher.Remove(name)
+				case <-ctx.Done():
 					return errDone
 				}
 				if err := handleRotate(); err != nil {
@@ -232,8 +245,7 @@ func followLogs(f *os.File, logWatcher *logger.LogWatcher, notifyRotate chan int
 				return errRetry
 			}
 			return err
-		case <-logWatcher.WatchClose():
-			fileWatcher.Remove(name)
+		case <-ctx.Done():
 			return errDone
 		}
 	}
@@ -290,7 +302,7 @@ func followLogs(f *os.File, logWatcher *logger.LogWatcher, notifyRotate chan int
 		}
 		select {
 		case logWatcher.Msg <- msg:
-		case <-logWatcher.WatchClose():
+		case <-ctx.Done():
 			logWatcher.Msg <- msg
 			for {
 				msg, err := decodeLogLine(dec, l)

  for _, pair := range tests {
-		go func() {
-			srv.Write(pair[0])
-		}()
+		go func(x []byte) {
+			srv.Write(x)
+		}(pair[0]) // write 复制的数据，避免race
 		n, err := l.Read(read)
```
