# 3. Reflector

## 3.1. Reflector

`Reflector`的主要作用是watch指定的k8s资源，并将变化同步到本地是`store`中。`Reflector`只会放置指定的`expectedType`类型的资源到`store`中，除非`expectedType`为nil。如果`resyncPeriod`不为零，那么`Reflector`为以`resyncPeriod`为周期定期执行list的操作，这样就可以使用`Reflector`来定期处理所有的对象，也可以逐步处理变化的对象。

常用属性说明：

- expectedType：期望放入缓存store的资源类型。
- store：watch的资源对应的本地缓存。
- listerWatcher：list和watch的接口。
- period：watch的周期，默认为1秒。
- resyncPeriod：resync的周期，当非零的时候，会按该周期执行list。
- lastSyncResourceVersion：最新一次看到的资源的版本号，主要在watch时候使用。

```go
// Reflector watches a specified resource and causes all changes to be reflected in the given store.
type Reflector struct {
	// name identifies this reflector. By default it will be a file:line if possible.
	name string
	// metrics tracks basic metric information about the reflector
	metrics *reflectorMetrics

	// The type of object we expect to place in the store.
	expectedType reflect.Type
	// The destination to sync up with the watch source
	store Store
	// listerWatcher is used to perform lists and watches.
	listerWatcher ListerWatcher
	// period controls timing between one watch ending and
	// the beginning of the next one.
	period       time.Duration
	resyncPeriod time.Duration
	ShouldResync func() bool
	// clock allows tests to manipulate time
	clock clock.Clock
	// lastSyncResourceVersion is the resource version token last
	// observed when doing a sync with the underlying store
	// it is thread safe, but not synchronized with the underlying store
	lastSyncResourceVersion string
	// lastSyncResourceVersionMutex guards read/write access to lastSyncResourceVersion
	lastSyncResourceVersionMutex sync.RWMutex
}
```

## 3.2. NewReflector

NewReflector主要用来构建Reflector的结构体。

> 此部分的代码位于/vendor/k8s.io/client-go/tools/cache/reflector.go

```go
// NewReflector creates a new Reflector object which will keep the given store up to
// date with the server's contents for the given resource. Reflector promises to
// only put things in the store that have the type of expectedType, unless expectedType
// is nil. If resyncPeriod is non-zero, then lists will be executed after every
// resyncPeriod, so that you can use reflectors to periodically process everything as
// well as incrementally processing the things that change.
func NewReflector(lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
	return NewNamedReflector(getDefaultReflectorName(internalPackages...), lw, expectedType, store, resyncPeriod)
}

// reflectorDisambiguator is used to disambiguate started reflectors.
// initialized to an unstable value to ensure meaning isn't attributed to the suffix.
var reflectorDisambiguator = int64(time.Now().UnixNano() % 12345)

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

## 3.3. Reflector.Run

Reflector.Run主要执行了`ListAndWatch`的方法。

```go
// Run starts a watch and handles watch events. Will restart the watch if it is closed.
// Run will exit when stopCh is closed.
func (r *Reflector) Run(stopCh <-chan struct{}) {
	glog.V(3).Infof("Starting reflector %v (%s) from %s", r.expectedType, r.resyncPeriod, r.name)
	wait.Until(func() {
		if err := r.ListAndWatch(stopCh); err != nil {
			utilruntime.HandleError(err)
		}
	}, r.period, stopCh)
}
```

## 3.4. ListAndWatch

ListAndWatch第一次会列出所有的对象，并获取资源对象的版本号，然后watch资源对象的版本号来查看是否有被变更。首先会将资源版本号设置为0，`list()`可能会导致本地的缓存相对于etcd里面的内容存在延迟，`Reflector`会通过`watch`的方法将延迟的部分补充上，使得本地的缓存数据与etcd的数据保持一致。

### 3.4.1. List

```go
// ListAndWatch first lists all items and get the resource version at the moment of call,
// and then use the resource version to watch.
// It returns error if ListAndWatch didn't even try to initialize watch.
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
	glog.V(3).Infof("Listing and watching %v from %s", r.expectedType, r.name)
	var resourceVersion string

	// Explicitly set "0" as resource version - it's fine for the List()
	// to be served from cache and potentially be delayed relative to
	// etcd contents. Reflector framework will catch up via Watch() eventually.
	options := metav1.ListOptions{ResourceVersion: "0"}
	r.metrics.numberOfLists.Inc()
	start := r.clock.Now()
	list, err := r.listerWatcher.List(options)
	if err != nil {
		return fmt.Errorf("%s: Failed to list %v: %v", r.name, r.expectedType, err)
	}
	r.metrics.listDuration.Observe(time.Since(start).Seconds())
	listMetaInterface, err := meta.ListAccessor(list)
	if err != nil {
		return fmt.Errorf("%s: Unable to understand list result %#v: %v", r.name, list, err)
	}
	resourceVersion = listMetaInterface.GetResourceVersion()
	items, err := meta.ExtractList(list)
	if err != nil {
		return fmt.Errorf("%s: Unable to understand list result %#v (%v)", r.name, list, err)
	}
	r.metrics.numberOfItemsInList.Observe(float64(len(items)))
	if err := r.syncWith(items, resourceVersion); err != nil {
		return fmt.Errorf("%s: Unable to sync list result: %v", r.name, err)
	}
	r.setLastSyncResourceVersion(resourceVersion)
    ...
}    
```

首先将资源的版本号设置为0，然后调用`listerWatcher.List(options)`，列出所有list的内容。

```go
// 版本号设置为0
options := metav1.ListOptions{ResourceVersion: "0"}
// list接口
list, err := r.listerWatcher.List(options)
```

获取资源版本号，并将list的内容提取成对象列表。

```go
// 获取版本号
resourceVersion = listMetaInterface.GetResourceVersion()
// 将list的内容提取成对象列表
items, err := meta.ExtractList(list)
```

将list中对象列表的内容和版本号存储到本地的缓存store中，并全量替换已有的store的内容。

```go
err := r.syncWith(items, resourceVersion)
```

syncWith调用了store的Replace的方法来替换原来store中的数据。

```go
// syncWith replaces the store's items with the given list.
func (r *Reflector) syncWith(items []runtime.Object, resourceVersion string) error {
	found := make([]interface{}, 0, len(items))
	for _, item := range items {
		found = append(found, item)
	}
	return r.store.Replace(found, resourceVersion)
}
```

Store.Replace方法定义如下：

```go
type Store interface {
	...
	// Replace will delete the contents of the store, using instead the
	// given list. Store takes ownership of the list, you should not reference
	// it after calling this function.
	Replace([]interface{}, string) error
    ...
}
```

最后设置最新的资源版本号。

```go
r.setLastSyncResourceVersion(resourceVersion)
```

setLastSyncResourceVersion:

```go
func (r *Reflector) setLastSyncResourceVersion(v string) {
	r.lastSyncResourceVersionMutex.Lock()
	defer r.lastSyncResourceVersionMutex.Unlock()
	r.lastSyncResourceVersion = v

	rv, err := strconv.Atoi(v)
	if err == nil {
		r.metrics.lastResourceVersion.Set(float64(rv))
	}
}
```

### 3.4.2. store.Resync

```go
resyncerrc := make(chan error, 1)
cancelCh := make(chan struct{})
defer close(cancelCh)
go func() {
	resyncCh, cleanup := r.resyncChan()
	defer func() {
		cleanup() // Call the last one written into cleanup
	}()
	for {
		select {
		case <-resyncCh:
		case <-stopCh:
			return
		case <-cancelCh:
			return
		}
		if r.ShouldResync == nil || r.ShouldResync() {
			glog.V(4).Infof("%s: forcing resync", r.name)
			if err := r.store.Resync(); err != nil {
				resyncerrc <- err
				return
			}
		}
		cleanup()
		resyncCh, cleanup = r.resyncChan()
	}
}()
```

核心代码：

```go
err := r.store.Resync()
```

store的具体对象为`DeltaFIFO`，即调用DeltaFIFO.Resync

```go
// Resync will send a sync event for each item
func (f *DeltaFIFO) Resync() error {
	f.lock.Lock()
	defer f.lock.Unlock()

	if f.knownObjects == nil {
		return nil
	}

	keys := f.knownObjects.ListKeys()
	for _, k := range keys {
		if err := f.syncKeyLocked(k); err != nil {
			return err
		}
	}
	return nil
}
```

### 3.4.3. Watch

```go
for {
	// give the stopCh a chance to stop the loop, even in case of continue statements further down on errors
	select {
	case <-stopCh:
		return nil
	default:
	}

	timemoutseconds := int64(minWatchTimeout.Seconds() * (rand.Float64() + 1.0))
	options = metav1.ListOptions{
		ResourceVersion: resourceVersion,
		// We want to avoid situations of hanging watchers. Stop any wachers that do not
		// receive any events within the timeout window.
		TimeoutSeconds: &timemoutseconds,
	}

	r.metrics.numberOfWatches.Inc()
	w, err := r.listerWatcher.Watch(options)
	if err != nil {
		switch err {
		case io.EOF:
			// watch closed normally
		case io.ErrUnexpectedEOF:
			glog.V(1).Infof("%s: Watch for %v closed with unexpected EOF: %v", r.name, r.expectedType, err)
		default:
			utilruntime.HandleError(fmt.Errorf("%s: Failed to watch %v: %v", r.name, r.expectedType, err))
		}
		// If this is "connection refused" error, it means that most likely apiserver is not responsive.
		// It doesn't make sense to re-list all objects because most likely we will be able to restart
		// watch where we ended.
		// If that's the case wait and resend watch request.
		if urlError, ok := err.(*url.Error); ok {
			if opError, ok := urlError.Err.(*net.OpError); ok {
				if errno, ok := opError.Err.(syscall.Errno); ok && errno == syscall.ECONNREFUSED {
					time.Sleep(time.Second)
					continue
				}
			}
		}
		return nil
	}

	if err := r.watchHandler(w, &resourceVersion, resyncerrc, stopCh); err != nil {
		if err != errorStopRequested {
			glog.Warningf("%s: watch of %v ended with: %v", r.name, r.expectedType, err)
		}
		return nil
	}
}
```

设置watch的超时时间，默认为5分钟。

```go
timemoutseconds := int64(minWatchTimeout.Seconds() * (rand.Float64() + 1.0))
options = metav1.ListOptions{
	ResourceVersion: resourceVersion,
	// We want to avoid situations of hanging watchers. Stop any wachers that do not
	// receive any events within the timeout window.
	TimeoutSeconds: &timemoutseconds,
}
```

执行listerWatcher.Watch(options)。

```go
w, err := r.listerWatcher.Watch(options)
```

执行watchHandler。

```go
err := r.watchHandler(w, &resourceVersion, resyncerrc, stopCh)
```

### 3.4.4. watchHandler

watchHandler主要是通过watch的方式保证当前的资源版本是最新的。

```go
// watchHandler watches w and keeps *resourceVersion up to date.
func (r *Reflector) watchHandler(w watch.Interface, resourceVersion *string, errc chan error, stopCh <-chan struct{}) error {
	start := r.clock.Now()
	eventCount := 0

	// Stopping the watcher should be idempotent and if we return from this function there's no way
	// we're coming back in with the same watch interface.
	defer w.Stop()
	// update metrics
	defer func() {
		r.metrics.numberOfItemsInWatch.Observe(float64(eventCount))
		r.metrics.watchDuration.Observe(time.Since(start).Seconds())
	}()

loop:
	for {
		select {
		case <-stopCh:
			return errorStopRequested
		case err := <-errc:
			return err
		case event, ok := <-w.ResultChan():
			if !ok {
				break loop
			}
			if event.Type == watch.Error {
				return apierrs.FromObject(event.Object)
			}
			if e, a := r.expectedType, reflect.TypeOf(event.Object); e != nil && e != a {
				utilruntime.HandleError(fmt.Errorf("%s: expected type %v, but watch event object had type %v", r.name, e, a))
				continue
			}
			meta, err := meta.Accessor(event.Object)
			if err != nil {
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
				continue
			}
			newResourceVersion := meta.GetResourceVersion()
			switch event.Type {
			case watch.Added:
				err := r.store.Add(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to add watch event object (%#v) to store: %v", r.name, event.Object, err))
				}
			case watch.Modified:
				err := r.store.Update(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to update watch event object (%#v) to store: %v", r.name, event.Object, err))
				}
			case watch.Deleted:
				// TODO: Will any consumers need access to the "last known
				// state", which is passed in event.Object? If so, may need
				// to change this.
				err := r.store.Delete(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to delete watch event object (%#v) from store: %v", r.name, event.Object, err))
				}
			default:
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
			}
			*resourceVersion = newResourceVersion
			r.setLastSyncResourceVersion(newResourceVersion)
			eventCount++
		}
	}

	watchDuration := r.clock.Now().Sub(start)
	if watchDuration < 1*time.Second && eventCount == 0 {
		r.metrics.numberOfShortWatches.Inc()
		return fmt.Errorf("very short watch: %s: Unexpected watch close - watch lasted less than a second and no items received", r.name)
	}
	glog.V(4).Infof("%s: Watch close - %v total %v items received", r.name, r.expectedType, eventCount)
	return nil
}
```

获取watch接口中的事件的channel，来获取事件的内容。

```go
for {
	select {
	...
	case event, ok := <-w.ResultChan():
    ...
}        
```

当获得添加、更新、删除的事件时，将对应的对象更新到本地缓存store中。

```go
switch event.Type {
case watch.Added:
	err := r.store.Add(event.Object)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("%s: unable to add watch event object (%#v) to store: %v", r.name, event.Object, err))
	}
case watch.Modified:
	err := r.store.Update(event.Object)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("%s: unable to update watch event object (%#v) to store: %v", r.name, event.Object, err))
	}
case watch.Deleted:
	// TODO: Will any consumers need access to the "last known
	// state", which is passed in event.Object? If so, may need
	// to change this.
	err := r.store.Delete(event.Object)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("%s: unable to delete watch event object (%#v) from store: %v", r.name, event.Object, err))
	}
default:
	utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
}
```

更新当前的最新版本号。

```go
newResourceVersion := meta.GetResourceVersion()
*resourceVersion = newResourceVersion
r.setLastSyncResourceVersion(newResourceVersion)
```

通过对Reflector模块的分析，可以看到多次使用到本地缓存store模块，而store的数据由DeltaFIFO赋值而来，以下针对DeltaFIFO和store做分析。

