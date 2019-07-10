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
