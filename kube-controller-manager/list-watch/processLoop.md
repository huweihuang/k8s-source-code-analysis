
# 6. processLoop

```go
func (c *controller) Run(stopCh <-chan struct{}) {
	...
	wait.Until(c.processLoop, time.Second, stopCh)
}
```

在controller.Run方法中会调用processLoop，以下分析`processLoop`的处理逻辑。

```go
// processLoop drains the work queue.
// TODO: Consider doing the processing in parallel. This will require a little thought
// to make sure that we don't end up processing the same object multiple times
// concurrently.
//
// TODO: Plumb through the stopCh here (and down to the queue) so that this can
// actually exit when the controller is stopped. Or just give up on this stuff
// ever being stoppable. Converting this whole package to use Context would
// also be helpful.
func (c *controller) processLoop() {
	for {
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
			if err == FIFOClosedError {
				return
			}
			if c.config.RetryOnError {
				// This is the safe way to re-enqueue.
				c.config.Queue.AddIfNotPresent(obj)
			}
		}
	}
}
```

processLoop主要处理任务队列中的任务，其中处理逻辑是调用具体的`ProcessFunc`函数来实现，核心代码为：

```go
obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
```

## 6.1. DeltaFIFO.Pop

Pop会阻塞住直到队列里面添加了新的对象，如果有多个对象，按照先进先出的原则处理，如果某个对象没有处理成功会重新被加入该队列中。

Pop中会调用具体的process函数来处理对象。

```go
// Pop blocks until an item is added to the queue, and then returns it.  If
// multiple items are ready, they are returned in the order in which they were
// added/updated. The item is removed from the queue (and the store) before it
// is returned, so if you don't successfully process it, you need to add it back
// with AddIfNotPresent().
// process function is called under lock, so it is safe update data structures
// in it that need to be in sync with the queue (e.g. knownKeys). The PopProcessFunc
// may return an instance of ErrRequeue with a nested error to indicate the current
// item should be requeued (equivalent to calling AddIfNotPresent under the lock).
//
// Pop returns a 'Deltas', which has a complete list of all the things
// that happened to the object (deltas) while it was sitting in the queue.
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
	f.lock.Lock()
	defer f.lock.Unlock()
	for {
		for len(f.queue) == 0 {
			// When the queue is empty, invocation of Pop() is blocked until new item is enqueued.
			// When Close() is called, the f.closed is set and the condition is broadcasted.
			// Which causes this loop to continue and return from the Pop().
			if f.IsClosed() {
				return nil, FIFOClosedError
			}

			f.cond.Wait()
		}
		id := f.queue[0]
		f.queue = f.queue[1:]
		item, ok := f.items[id]
		if f.initialPopulationCount > 0 {
			f.initialPopulationCount--
		}
		if !ok {
			// Item may have been deleted subsequently.
			continue
		}
		delete(f.items, id)
		err := process(item)
		if e, ok := err.(ErrRequeue); ok {
			f.addIfNotPresent(id, item)
			err = e.Err
		}
		// Don't need to copyDeltas here, because we're transferring
		// ownership to the caller.
		return item, err
	}
}
```

核心代码：

```go
for {
	...
	item, ok := f.items[id]
	...
	err := process(item)
	if e, ok := err.(ErrRequeue); ok {
		f.addIfNotPresent(id, item)
		err = e.Err
	}
	// Don't need to copyDeltas here, because we're transferring
	// ownership to the caller.
	return item, err
}
```

## 6.2. HandleDeltas

```go
cfg := &Config{
	Queue:            fifo,
	ListerWatcher:    s.listerWatcher,
	ObjectType:       s.objectType,
	FullResyncPeriod: s.resyncCheckPeriod,
	RetryOnError:     false,
	ShouldResync:     s.processor.shouldResync,

	Process: s.HandleDeltas,
}
```

其中process函数就是在sharedIndexInformer.Run方法中，给config.Process赋值的`HandleDeltas`函数。

```go
func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	// from oldest to newest
	for _, d := range obj.(Deltas) {
		switch d.Type {
		case Sync, Added, Updated:
			isSync := d.Type == Sync
			s.cacheMutationDetector.AddObject(d.Object)
			if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
				if err := s.indexer.Update(d.Object); err != nil {
					return err
				}
				s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
			} else {
				if err := s.indexer.Add(d.Object); err != nil {
					return err
				}
				s.processor.distribute(addNotification{newObj: d.Object}, isSync)
			}
		case Deleted:
			if err := s.indexer.Delete(d.Object); err != nil {
				return err
			}
			s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
		}
	}
	return nil
}
```

核心代码：

```go
switch d.Type {
case Sync, Added, Updated:
	...
	if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
		...
		s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
	} else {
		...
		s.processor.distribute(addNotification{newObj: d.Object}, isSync)
	}
case Deleted:
	...
	s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
}
```

根据不同的类型，调用`processor.distribute`方法，该方法将对象加入`processorListener`的channel中。

## 6.3. sharedProcessor.distribute

```go
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	if sync {
		for _, listener := range p.syncingListeners {
			listener.add(obj)
		}
	} else {
		for _, listener := range p.listeners {
			listener.add(obj)
		}
	}
}
```

**processorListener.add:**

```go
func (p *processorListener) add(notification interface{}) {
	p.addCh <- notification
}
```

综合以上的分析，可以看出processLoop通过调用HandleDeltas，再调用distribute，processorListener.add最终将不同更新类型的对象加入`processorListener`的channel中，供processorListener.Run使用。以下分析processorListener.Run的部分。

# 7. processor

processor的主要功能就是记录了所有的回调函数实例(即 ResourceEventHandler 实例)，并负责触发这些函数。在sharedIndexInformer.Run部分会调用processor.run。

流程：

1. listenser的add函数负责将notify装进pendingNotifications。
2. pop函数取出pendingNotifications的第一个nofify,输出到nextCh channel。
3. run函数则负责取出notify，然后根据notify的类型(增加、删除、更新)触发相应的处理函数，这些函数是在不同的`NewXxxcontroller`实现中注册的。

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	...
	wg.StartWithChannel(processorStopCh, s.processor.run)
	...
}
```

## 7.1. sharedProcessor.Run

```go
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
   func() {
      p.listenersLock.RLock()
      defer p.listenersLock.RUnlock()
      for _, listener := range p.listeners {
         p.wg.Start(listener.run)
         p.wg.Start(listener.pop)
      }
   }()
   <-stopCh
   p.listenersLock.RLock()
   defer p.listenersLock.RUnlock()
   for _, listener := range p.listeners {
      close(listener.addCh) // Tell .pop() to stop. .pop() will tell .run() to stop
   }
   p.wg.Wait() // Wait for all .pop() and .run() to stop
}
```

### 7.1.1. listener.pop

pop函数取出pendingNotifications的第一个nofify,输出到nextCh channel。

```go
func (p *processorListener) pop() {
	defer utilruntime.HandleCrash()
	defer close(p.nextCh) // Tell .run() to stop

	var nextCh chan<- interface{}
	var notification interface{}
	for {
		select {
		case nextCh <- notification:
			// Notification dispatched
			var ok bool
			notification, ok = p.pendingNotifications.ReadOne()
			if !ok { // Nothing to pop
				nextCh = nil // Disable this select case
			}
		case notificationToAdd, ok := <-p.addCh:
			if !ok {
				return
			}
			if notification == nil { // No notification to pop (and pendingNotifications is empty)
				// Optimize the case - skip adding to pendingNotifications
				notification = notificationToAdd
				nextCh = p.nextCh
			} else { // There is already a notification waiting to be dispatched
				p.pendingNotifications.WriteOne(notificationToAdd)
			}
		}
	}
}
```

### 7.1.2. listener.run

 listener.run部分根据不同的更新类型调用不同的处理函数。

```go
func (p *processorListener) run() {
	defer utilruntime.HandleCrash()

	for next := range p.nextCh {
		switch notification := next.(type) {
		case updateNotification:
			p.handler.OnUpdate(notification.oldObj, notification.newObj)
		case addNotification:
			p.handler.OnAdd(notification.newObj)
		case deleteNotification:
			p.handler.OnDelete(notification.oldObj)
		default:
			utilruntime.HandleError(fmt.Errorf("unrecognized notification: %#v", next))
		}
	}
}
```

其中具体的实现函数handler是在NewDeploymentController（其他不同类型的controller类似）中赋值的，而该handler是一个接口，具体如下：

```go
// ResourceEventHandler can handle notifications for events that happen to a
// resource. The events are informational only, so you can't return an
// error.
//  * OnAdd is called when an object is added.
//  * OnUpdate is called when an object is modified. Note that oldObj is the
//      last known state of the object-- it is possible that several changes
//      were combined together, so you can't use this to see every single
//      change. OnUpdate is also called when a re-list happens, and it will
//      get called even if nothing changed. This is useful for periodically
//      evaluating or syncing something.
//  * OnDelete will get the final state of the item if it is known, otherwise
//      it will get an object of type DeletedFinalStateUnknown. This can
//      happen if the watch is closed and misses the delete event and we don't
//      notice the deletion until the subsequent re-list.
type ResourceEventHandler interface {
	OnAdd(obj interface{})
	OnUpdate(oldObj, newObj interface{})
	OnDelete(obj interface{})
}
```

## 7.2. ResourceEventHandler

以下以DeploymentController的处理逻辑为例。

在`NewDeploymentController`部分会注册deployment的事件函数，以下注册了三种类型的事件函数，其中包括：dInformer、rsInformer和podInformer。

```go
// NewDeploymentController creates a new DeploymentController.
func NewDeploymentController(dInformer extensionsinformers.DeploymentInformer, rsInformer extensionsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, client clientset.Interface) (*DeploymentController, error) {
	...
	dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addDeployment,
		UpdateFunc: dc.updateDeployment,
		// This will enter the sync loop and no-op, because the deployment has been deleted from the store.
		DeleteFunc: dc.deleteDeployment,
	})
	rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addReplicaSet,
		UpdateFunc: dc.updateReplicaSet,
		DeleteFunc: dc.deleteReplicaSet,
	})
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		DeleteFunc: dc.deletePod,
	})
    ...
}    
```

### 7.2.1. addDeployment

以下以`addDeployment`为例，addDeployment主要是将对象加入到enqueueDeployment的队列中。

```go
func (dc *DeploymentController) addDeployment(obj interface{}) {
	d := obj.(*extensions.Deployment)
	glog.V(4).Infof("Adding deployment %s", d.Name)
	dc.enqueueDeployment(d)
}
```

enqueueDeployment的定义

```go
type DeploymentController struct {
	...
	enqueueDeployment func(deployment *extensions.Deployment)
    ...
}    
```

将dc.enqueue赋值给dc.enqueueDeployment

```go
dc.enqueueDeployment = dc.enqueue
```

dc.enqueue调用了dc.queue.Add(key)

```go
func (dc *DeploymentController) enqueue(deployment *extensions.Deployment) {
	key, err := controller.KeyFunc(deployment)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Couldn't get key for object %#v: %v", deployment, err))
		return
	}

	dc.queue.Add(key)
}
```

dc.queue主要记录了需要被同步的deployment的对象，供syncDeployment使用。

```go
dc := &DeploymentController{
	...
	queue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "deployment"),
}
```

NewNamedRateLimitingQueue

```go
func NewNamedRateLimitingQueue(rateLimiter RateLimiter, name string) RateLimitingInterface {
	return &rateLimitingType{
		DelayingInterface: NewNamedDelayingQueue(name),
		rateLimiter:       rateLimiter,
	}
}
```

通过以上分析，可以看出processor记录了不同类似的事件函数，其中事件函数在NewXxxController构造函数部分注册，具体事件函数的处理，一般是将需要处理的对象加入对应的controller的任务队列中，然后由类似syncDeployment的同步函数来维持期望状态的同步逻辑。
