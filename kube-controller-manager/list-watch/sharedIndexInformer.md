# 1. sharedInformerFactory.Start

在controller-manager的Run函数部分调用了InformerFactory.Start的方法。

> 此部分代码位于/cmd/kube-controller-manager/app/controllermanager.go

```go
// Run runs the KubeControllerManagerOptions.  This should never exit.
func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {
    ...
		controllerContext.InformerFactory.Start(controllerContext.Stop)
		close(controllerContext.InformersStarted)
    ...
}
```

InformerFactory是一个`SharedInformerFactory`的接口，接口定义如下：

> 此部分代码位于vendor/k8s.io/client-go/informers/internalinterfaces/factory_interfaces.go

```go
// SharedInformerFactory a small interface to allow for adding an informer without an import cycle
type SharedInformerFactory interface {
	Start(stopCh <-chan struct{})
	InformerFor(obj runtime.Object, newFunc NewInformerFunc) cache.SharedIndexInformer
}

```

Start方法初始化各种类型的informer，并且每个类型起了个informer.Run的goroutine。

> 此部分代码位于vendor/k8s.io/client-go/informers/factory.go

```go
// Start initializes all requested informers.
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
	f.lock.Lock()
	defer f.lock.Unlock()

	for informerType, informer := range f.informers {
		if !f.startedInformers[informerType] {
			go informer.Run(stopCh)
			f.startedInformers[informerType] = true
		}
	}
}
```

# 2. sharedIndexInformer.Run

> 此部分的代码位于/vendor/k8s.io/client-go/tools/cache/shared_informer.go

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()

	fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, nil, s.indexer)

	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    s.listerWatcher,
		ObjectType:       s.objectType,
		FullResyncPeriod: s.resyncCheckPeriod,
		RetryOnError:     false,
		ShouldResync:     s.processor.shouldResync,

		Process: s.HandleDeltas,
	}

	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()

		s.controller = New(cfg)
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

	// Separate stop channel because Processor should be stopped strictly after controller
	processorStopCh := make(chan struct{})
	var wg wait.Group
	defer wg.Wait()              // Wait for Processor to stop
	defer close(processorStopCh) // Tell Processor to stop
	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
	wg.StartWithChannel(processorStopCh, s.processor.run)

	defer func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		s.stopped = true // Don't want any new listeners
	}()
	s.controller.Run(stopCh)
}
```

## 2.1. NewDeltaFIFO

DeltaFIFO是一个对象变化的存储队列，依据先进先出的原则，process的函数接收该队列的Pop方法的输出对象来处理相关功能。

```go
fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, nil, s.indexer)
```

## 2.2. Config

构造controller的配置文件，构造process，即HandleDeltas，该函数为后面使用到的process函数。

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

## 2.3. controller

调用New(cfg)，构建sharedIndexInformer的controller。

```go
func() {
	s.startedLock.Lock()
	defer s.startedLock.Unlock()

	s.controller = New(cfg)
	s.controller.(*controller).clock = s.clock
	s.started = true
}()
```

## 2.4. cacheMutationDetector.Run

调用s.cacheMutationDetector.Run，检查缓存对象是否变化。

```go
wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
```

**defaultCacheMutationDetector.Run**

```go
func (d *defaultCacheMutationDetector) Run(stopCh <-chan struct{}) {
	// we DON'T want protection from panics.  If we're running this code, we want to die
	for {
		d.CompareObjects()

		select {
		case <-stopCh:
			return
		case <-time.After(d.period):
		}
	}
}
```

**CompareObjects**

```go
func (d *defaultCacheMutationDetector) CompareObjects() {
	d.lock.Lock()
	defer d.lock.Unlock()

	altered := false
	for i, obj := range d.cachedObjs {
		if !reflect.DeepEqual(obj.cached, obj.copied) {
			fmt.Printf("CACHE %s[%d] ALTERED!\n%v\n", d.name, i, diff.ObjectDiff(obj.cached, obj.copied))
			altered = true
		}
	}

	if altered {
		msg := fmt.Sprintf("cache %s modified", d.name)
		if d.failureFunc != nil {
			d.failureFunc(msg)
			return
		}
		panic(msg)
	}
}
```

## 2.5. processor.run

调用s.processor.run，将调用sharedProcessor.run，会调用Listener.run和Listener.pop,执行处理queue的函数。

```go
wg.StartWithChannel(processorStopCh, s.processor.run)
```

**sharedProcessor.Run**

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

该部分逻辑待后面分析。

## 2.6. controller.Run

调用s.controller.Run，构建Reflector，进行对etcd的缓存

```go
defer func() {
	s.startedLock.Lock()
	defer s.startedLock.Unlock()
	s.stopped = true // Don't want any new listeners
}()
s.controller.Run(stopCh)
```

controller.Run

> 此部分代码位于/vendor/k8s.io/client-go/tools/cache/controller.go

```go
// Run begins processing items, and will continue until a value is sent down stopCh.
// It's an error to call Run more than once.
// Run blocks; call via go.
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	go func() {
		<-stopCh
		c.config.Queue.Close()
	}()
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
	r.ShouldResync = c.config.ShouldResync
	r.clock = c.clock

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group
	defer wg.Wait()

	wg.StartWithChannel(stopCh, r.Run)

	wait.Until(c.processLoop, time.Second, stopCh)
}
```

核心代码：

```go
// 构建Reflector
r := NewReflector(
	c.config.ListerWatcher,
	c.config.ObjectType,
	c.config.Queue,
	c.config.FullResyncPeriod,
)
// 运行Reflector
wg.StartWithChannel(stopCh, r.Run)
// 执行processLoop
wait.Until(c.processLoop, time.Second, stopCh)
```
