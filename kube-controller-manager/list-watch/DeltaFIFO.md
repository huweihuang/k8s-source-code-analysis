
# 4. DeltaFIFO

DeltaFIFO由NewDeltaFIFO初始化，并赋值给config.Queue。

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, nil, s.indexer)

	cfg := &Config{
		Queue:            fifo,
		...
	}
    ...
}    
```

## 4.1. NewDeltaFIFO

```go
// NewDeltaFIFO returns a Store which can be used process changes to items.
//
// keyFunc is used to figure out what key an object should have. (It's
// exposed in the returned DeltaFIFO's KeyOf() method, with bonus features.)
//
// 'compressor' may compress as many or as few items as it wants
// (including returning an empty slice), but it should do what it
// does quickly since it is called while the queue is locked.
// 'compressor' may be nil if you don't want any delta compression.
//
// 'keyLister' is expected to return a list of keys that the consumer of
// this queue "knows about". It is used to decide which items are missing
// when Replace() is called; 'Deleted' deltas are produced for these items.
// It may be nil if you don't need to detect all deletions.
// TODO: consider merging keyLister with this object, tracking a list of
//       "known" keys when Pop() is called. Have to think about how that
//       affects error retrying.
// TODO(lavalamp): I believe there is a possible race only when using an
//                 external known object source that the above TODO would
//                 fix.
//
// Also see the comment on DeltaFIFO.
func NewDeltaFIFO(keyFunc KeyFunc, compressor DeltaCompressor, knownObjects KeyListerGetter) *DeltaFIFO {
	f := &DeltaFIFO{
		items:           map[string]Deltas{},
		queue:           []string{},
		keyFunc:         keyFunc,
		deltaCompressor: compressor,
		knownObjects:    knownObjects,
	}
	f.cond.L = &f.lock
	return f
}
```

controller.Run的部分调用了NewReflector。

```go
func (c *controller) Run(stopCh <-chan struct{}) {
	...
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
    ...
}    
```

NewReflector构造函数，将c.config.Queue赋值给Reflector.store的属性。

```go
func NewReflector(lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
	return NewNamedReflector(getDefaultReflectorName(internalPackages...), lw, expectedType, store, resyncPeriod)
}

// NewNamedReflector same as NewReflector, but with a specified name for logging
func NewNamedReflector(name string, lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
	reflectorSuffix := atomic.AddInt64(&reflectorDisambiguator, 1)
	r := &Reflector{
		name: name,
		// we need this to be unique per process (some names are still the same)but obvious who it belongs to
		metrics:       newReflectorMetrics(makeValidPromethusMetricLabel(fmt.Sprintf("reflector_"+name+"_%d", reflectorSuffix))),
		listerWatcher: lw,
		store:         store,
		expectedType:  reflect.TypeOf(expectedType),
		period:        time.Second,
		resyncPeriod:  resyncPeriod,
		clock:         &clock.RealClock{},
	}
	return r
}
```

## 4.2. DeltaFIFO

DeltaFIFO是一个生产者与消费者的队列，其中Reflector是生产者，消费者调用Pop()的方法。

DeltaFIFO主要用在以下场景：

- 希望对象变更最多处理一次
- 处理对象时，希望查看自上次处理对象以来发生的所有事情
- 要处理对象的删除
- 希望定期重新处理对象

```go
// DeltaFIFO is like FIFO, but allows you to process deletes.
//
// DeltaFIFO is a producer-consumer queue, where a Reflector is
// intended to be the producer, and the consumer is whatever calls
// the Pop() method.
//
// DeltaFIFO solves this use case:
//  * You want to process every object change (delta) at most once.
//  * When you process an object, you want to see everything
//    that's happened to it since you last processed it.
//  * You want to process the deletion of objects.
//  * You might want to periodically reprocess objects.
//
// DeltaFIFO's Pop(), Get(), and GetByKey() methods return
// interface{} to satisfy the Store/Queue interfaces, but it
// will always return an object of type Deltas.
//
// A note on threading: If you call Pop() in parallel from multiple
// threads, you could end up with multiple threads processing slightly
// different versions of the same object.
//
// A note on the KeyLister used by the DeltaFIFO: It's main purpose is
// to list keys that are "known", for the purpose of figuring out which
// items have been deleted when Replace() or Delete() are called. The deleted
// object will be included in the DeleteFinalStateUnknown markers. These objects
// could be stale.
//
// You may provide a function to compress deltas (e.g., represent a
// series of Updates as a single Update).
type DeltaFIFO struct {
	// lock/cond protects access to 'items' and 'queue'.
	lock sync.RWMutex
	cond sync.Cond

	// We depend on the property that items in the set are in
	// the queue and vice versa, and that all Deltas in this
	// map have at least one Delta.
	items map[string]Deltas
	queue []string

	// populated is true if the first batch of items inserted by Replace() has been populated
	// or Delete/Add/Update was called first.
	populated bool
	// initialPopulationCount is the number of items inserted by the first call of Replace()
	initialPopulationCount int

	// keyFunc is used to make the key used for queued item
	// insertion and retrieval, and should be deterministic.
	keyFunc KeyFunc

	// deltaCompressor tells us how to combine two or more
	// deltas. It may be nil.
	deltaCompressor DeltaCompressor

	// knownObjects list keys that are "known", for the
	// purpose of figuring out which items have been deleted
	// when Replace() or Delete() is called.
	knownObjects KeyListerGetter

	// Indication the queue is closed.
	// Used to indicate a queue is closed so a control loop can exit when a queue is empty.
	// Currently, not used to gate any of CRED operations.
	closed     bool
	closedLock sync.Mutex
}
```



## 4.3. Queue & Store

DeltaFIFO的类型是Queue接口，Reflector.store是Store接口，Queue接口是一个存储队列，Process的方法执行Queue.Pop出来的数据对象，

```go
// Queue is exactly like a Store, but has a Pop() method too.
type Queue interface {
	Store

	// Pop blocks until it has something to process.
	// It returns the object that was process and the result of processing.
	// The PopProcessFunc may return an ErrRequeue{...} to indicate the item
	// should be requeued before releasing the lock on the queue.
	Pop(PopProcessFunc) (interface{}, error)

	// AddIfNotPresent adds a value previously
	// returned by Pop back into the queue as long
	// as nothing else (presumably more recent)
	// has since been added.
	AddIfNotPresent(interface{}) error

	// Return true if the first batch of items has been popped
	HasSynced() bool

	// Close queue
	Close()
}
```
