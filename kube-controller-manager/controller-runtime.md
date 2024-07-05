---
title: "controller-runtime源码分析"
linkTitle: "controller-runtime源码分析"
weight: 10
catalog: true
date: 2024-6-10 21:24:24
subtitle:
header-img: "https://res.cloudinary.com/dqxtn0ick/image/upload/v1542285471/header/building.jpg"
tags:
- 源码分析
catagories:
- 源码分析
---

> 本文主要分析controller-runtime的源码，源码版本为v0.16.3

# 1. 概述

controller-runtime源码地址：https://github.com/kubernetes-sigs/controller-runtime。

controller-runtime项目是一个用于快速构建k8s operator的工具包。其中kubebuilder和operator-sdk项目都是通过controller-runtime项目来快速编写k8s operator的工具。

本文以kubebuilder的代码生成架构为例，分析controller-runtime的逻辑。kubebuilder框架生成的代码参考：[https://github.com/huweihuang/venus](https://github.com/huweihuang/venus)

# 2. controller-runtime架构图

![](https://res.cloudinary.com/dqxtn0ick/image/upload/v1700367468/article/kubernetes/kubebuilder/controller_runtime_overview.jpg)

代码目录：

```bash
pkg
├── builder
├── cache
├── client  # client用于操作k8s的对象
├── cluster
├── config
├── controller  # controller逻辑
├── envtest
├── event
├── finalizer
├── handler
├── internal  # 核心代码 controller的具体实现
├── leaderelection
├── manager   # 核心代码
├── predicate
├── ratelimiter
├── reconcile
├── recorder
├── scheme
├── source
└── webhook
```

# 3. Operator框架逻辑

> 代码参考：https://github.com/huweihuang/venus/blob/main/cmd/app/operator.go#L71

operator代码框架的主体逻辑包括以下几个部分。

- `manager`：主要用来管理多个的controller，构建，注册，运行controller。

- `controller`：主要用来封装reconciler的控制器。

- `reconciler`：具体执行业务逻辑的函数。

manger的框架主要包含以下几个部分。

- mgr:=ctrl.NewManager：构建一个manager对象。

- Reconciler.SetupWithManager(mgr)：注册controller到manager对象。

- mgr.Start(ctrl.SetupSignalHandler())：运行manager从而运行controller的逻辑。

代码如下：

```go
  // 构建manager对象，主要的初始化参数包括
	// - kubeconfig
	// - controller的option参数
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:                 scheme,
        Metrics:                metricsserver.Options{BindAddress: opt.MetricsAddr},
        HealthProbeBindAddress: opt.ProbeAddr,
        LeaderElection:         opt.EnableLeaderElection,
        LeaderElectionID:       "52609143.huweihuang.com",
        Controller: config.Controller{
            MaxConcurrentReconciles: opt.MaxConcurrentReconciles,
        },
    })

	// 将controller注册到manager中，并初始化controller对象。
    if err = (&venuscontroller.RedisReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create controller", "controller", "Redis")
        return err
    }

	// 运行manager对象，从而运行controller中的reconcile逻辑。
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        return err
    }
```

# 4. NewManager

NewManager初始化一个manager 用来管理和创建controller对象。一个manager可以关联多个controller对象。manager是一个接口，而最终是实现结构体是`controllerManager`的对象。

## 4.1. Manager接口

```go
type Manager interface {
	// Cluster holds a variety of methods to interact with a cluster.
	cluster.Cluster

	// Add will set requested dependencies on the component, and cause the component to be
	// started when Start is called.
	// Depending on if a Runnable implements LeaderElectionRunnable interface, a Runnable can be run in either
	// non-leaderelection mode (always running) or leader election mode (managed by leader election if enabled).
  
  // 通过Runnable接口将具体的controller注册到manager中。
	Add(Runnable) error

	// Start starts all registered Controllers and blocks until the context is cancelled.
	// Returns an error if there is an error starting any controller.
	//
	// If LeaderElection is used, the binary must be exited immediately after this returns,
	// otherwise components that need leader election might continue to run after the leader
	// lock was lost.
  
  // 运行具体的逻辑
	Start(ctx context.Context) error
}
```

## 4.2. NewControllerManager

New构建一个具体的controllerManager的对象。

```go
func New(config *rest.Config, options Options) (Manager, error) {
    // Set default values for options fields
    options = setOptionsDefaults(options)
    ... 

    errChan := make(chan error)
    runnables := newRunnables(options.BaseContext, errChan)
    return &controllerManager{
        stopProcedureEngaged:          pointer.Int64(0),
        cluster:                       cluster,
        runnables:                     runnables,
        errChan:                       errChan,
        recorderProvider:              recorderProvider,
        resourceLock:                  resourceLock,
        metricsServer:                 metricsServer,
        controllerConfig:              options.Controller,
        logger:                        options.Logger,
        elected:                       make(chan struct{}),
        webhookServer:                 options.WebhookServer,
        leaderElectionID:              options.LeaderElectionID,
        leaseDuration:                 *options.LeaseDuration,
        renewDeadline:                 *options.RenewDeadline,
        retryPeriod:                   *options.RetryPeriod,
        healthProbeListener:           healthProbeListener,
        readinessEndpointName:         options.ReadinessEndpointName,
        livenessEndpointName:          options.LivenessEndpointName,
        pprofListener:                 pprofListener,
        gracefulShutdownTimeout:       *options.GracefulShutdownTimeout,
        internalProceduresStop:        make(chan struct{}),
        leaderElectionStopped:         make(chan struct{}),
        leaderElectionReleaseOnCancel: options.LeaderElectionReleaseOnCancel,
    }, nil
}
```

# 5. SetupWithManager

SetupWithManager将具体的controller注册到manager中。其中通过`Complete`完成controller的初始化。

```go
// SetupWithManager sets up the controller with the Manager.
func (r *RedisReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&venusv1.Redis{}).
        Complete(r)
}
```

SetupWithManager通过NewControllerManagedBy方法构建了一个Builder的对象。

```go
// Builder builds a Controller.
type Builder struct {
    forInput         ForInput
    ownsInput        []OwnsInput
    watchesInput     []WatchesInput
    mgr              manager.Manager
    globalPredicates []predicate.Predicate
    ctrl             controller.Controller
    ctrlOptions      controller.Options
    name             string
}

// ControllerManagedBy returns a new controller builder that will be started by the provided Manager.
func ControllerManagedBy(m manager.Manager) *Builder {
    return &Builder{mgr: m}
}
```

通过builder对象完成controller的初始化。

## 5.1. controller初始化

```go
// Build builds the Application Controller and returns the Controller it created.
func (blder *Builder) Build(r reconcile.Reconciler) (controller.Controller, error) {

    // Set the ControllerManagedBy
    if err := blder.doController(r); err != nil {  // 初始化controller
        return nil, err
    }

    // Set the Watch
    if err := blder.doWatch(); err != nil {   // 添加event handler
        return nil, err
    }

    return blder.ctrl, nil
}
```

`doController`最终通过调用`NewUnmanaged`构建一个controller对象。并传入自定义的reconciler对象。

```go
func New(name string, mgr manager.Manager, options Options) (Controller, error) {
    c, err := NewUnmanaged(name, mgr, options)
    ...
    // Add the controller as a Manager components
    return c, mgr.Add(c)
}

// NewUnmanaged returns a new controller without adding it to the manager. The
// caller is responsible for starting the returned controller.
func NewUnmanaged(name string, mgr manager.Manager, options Options) (Controller, error) {
    if options.Reconciler == nil {
        return nil, fmt.Errorf("must specify Reconciler")
    }
    ...
    // Create controller with dependencies set
    return &controller.Controller{
        Do: options.Reconciler,   // 将具体的reconciler函数传递到controller的reconciler。
        MakeQueue: func() workqueue.RateLimitingInterface {
            return workqueue.NewRateLimitingQueueWithConfig(options.RateLimiter, workqueue.RateLimitingQueueConfig{
                Name: name,
            })
        }, // 初始化任务队列
        MaxConcurrentReconciles: options.MaxConcurrentReconciles, // 设置controller的并发数
        CacheSyncTimeout:        options.CacheSyncTimeout,
        Name:                    name,
        LogConstructor:          options.LogConstructor,
        RecoverPanic:            options.RecoverPanic,
        LeaderElected:           options.NeedLeaderElection,
    }, nil
```

## 5.2. 添加event handler

`doWatch`最终会运行informer.start添加event handler。

```go
// Watch implements controller.Controller.
func (c *Controller) Watch(src source.Source, evthdler handler.EventHandler, prct ...predicate.Predicate) error {
    c.mu.Lock()
    defer c.mu.Unlock()

    // Controller hasn't started yet, store the watches locally and return.
    //
    // These watches are going to be held on the controller struct until the manager or user calls Start(...).
    if !c.Started {
        c.startWatches = append(c.startWatches, watchDescription{src: src, handler: evthdler, predicates: prct})
        return nil
    }

    c.LogConstructor(nil).Info("Starting EventSource", "source", src)
    return src.Start(c.ctx, evthdler, c.Queue, prct...)
}
```

add event handler

```go
func (is *Informer) Start(ctx context.Context, handler handler.EventHandler, queue workqueue.RateLimitingInterface,
    prct ...predicate.Predicate) error {
    // Informer should have been specified by the user.
    if is.Informer == nil {
        return fmt.Errorf("must specify Informer.Informer")
    }

    _, err := is.Informer.AddEventHandler(internal.NewEventHandler(ctx, queue, handler, prct).HandlerFuncs())
    if err != nil {
        return err
    }
    return nil
}
```

# 6. mgr.Start

controllerManager运行之前注册的runnables的函数，其中包括controller的函数。

```go
func (cm *controllerManager) Start(ctx context.Context) (err error) {
	// Start and wait for caches.
	if err := cm.runnables.Caches.Start(cm.internalCtx); err != nil {
	}

	// Start the non-leaderelection Runnables after the cache has synced.
	if err := cm.runnables.Others.Start(cm.internalCtx); err != nil {
	}

	// Start the leader election and all required runnables.
	{
		ctx, cancel := context.WithCancel(context.Background())
		cm.leaderElectionCancel = cancel
		go func() {
			if cm.resourceLock != nil {
				if err := cm.startLeaderElection(ctx); err != nil {
					cm.errChan <- err
				}
			} else {
				// Treat not having leader election enabled the same as being elected.
				if err := cm.startLeaderElectionRunnables(); err != nil {
					cm.errChan <- err
				}
				close(cm.elected)
			}
		}()
	}
	...
}
```

## 6.1. controller.start

start主要包含2个部分

- 同步cache：WaitForSync
- 启动指定并发数的worker：processNextWorkItem

该部分的代码逻辑跟k8s controller-manager中的具体的controller的逻辑类似。

```go
func (c *Controller) Start(ctx context.Context) error {
  ...
	err := func() error {
		...
		for _, watch := range c.startWatches {
			syncingSource, ok := watch.src.(source.SyncingSource)
      // 同步list-watch中的cache数据。
			if err := func() error {
				if err := syncingSource.WaitForSync(sourceStartCtx); err != nil {
          ...
				}
			}
		}

    // 运行指定并发数的processNextWorkItem任务。
		// Launch workers to process resources
		c.LogConstructor(nil).Info("Starting workers", "worker count", c.MaxConcurrentReconciles)
		wg.Add(c.MaxConcurrentReconciles)
		for i := 0; i < c.MaxConcurrentReconciles; i++ {
			go func() {
				defer wg.Done()
				// Run a worker thread that just dequeues items, processes them, and marks them done.
				// It enforces that the reconcileHandler is never invoked concurrently with the same object.
				for c.processNextWorkItem(ctx) {
				}
			}()
		}
	...
}
```

## 6.2. processNextWorkItem

经典的`processNextWorkItem`函数，最终调用`reconcileHandler`来处理具体的逻辑。

```go
func (c *Controller) processNextWorkItem(ctx context.Context) bool {
	obj, shutdown := c.Queue.Get()
	if shutdown {
		// Stop working
		return false
	}

	defer c.Queue.Done(obj)

	ctrlmetrics.ActiveWorkers.WithLabelValues(c.Name).Add(1)
	defer ctrlmetrics.ActiveWorkers.WithLabelValues(c.Name).Add(-1)

	c.reconcileHandler(ctx, obj)
	return true
}
```

# 7. reconcileHandler

reconcileHandler部分的代码是整个reconciler逻辑中的核心，自定义的reconciler函数最终是调用了reconcileHandler来实现，并且该函数描述了具体的任务队列处理的几种类型。

- `err != nil`：如果错误不为空，则重新入队，等待处理。
- `result.RequeueAfter > 0`：如果指定RequeueAfter > 0，则做延迟入队处理。
- `result.Requeue`：如何指定了requeue则表示马上重新入队处理。
- `err == nil` : 如果错误为空，表示reconcile成功，则移除队列的任务。

```go
func (c *Controller) reconcileHandler(ctx context.Context, obj interface{}) {
	// 获取k8s crd的具体对象
	req, ok := obj.(reconcile.Request)
	if !ok {
		// As the item in the workqueue is actually invalid, we call
		// Forget here else we'd go into a loop of attempting to
		// process a work item that is invalid.
		c.Queue.Forget(obj)
		c.LogConstructor(nil).Error(nil, "Queue item was not a Request", "type", fmt.Sprintf("%T", obj), "value", obj)
		// Return true, don't take a break
		return
	}

	// 调用用户定义的Reconcile函数，并对返回结果进行处理。
	result, err := c.Reconcile(ctx, req)
	switch {
	case err != nil:
    // 如果错误不为空，则重新入队，等待处理
		if errors.Is(err, reconcile.TerminalError(nil)) {
			ctrlmetrics.TerminalReconcileErrors.WithLabelValues(c.Name).Inc()
		} else {
			c.Queue.AddRateLimited(req)
		}
	case result.RequeueAfter > 0:
		// 如果指定了延迟入队，则做延迟入队处理
		c.Queue.Forget(obj)
		c.Queue.AddAfter(req, result.RequeueAfter)
	case result.Requeue:
		// 如果指定了马上入队，则做相应处理
		c.Queue.AddRateLimited(req)
	default:
    // 如果错误为空，表示reconcile成功，则移除队列的任务
		log.V(5).Info("Reconcile successful")
		// Finally, if no error occurs we Forget this item so it does not
		// get queued again until another change happens.
		c.Queue.Forget(obj)
	}
}
```

# 8. 总结

1. controller-runtime封装了k8s-controller-manager控制器的主要逻辑，其中就包括创建list-watch对象，waitForSync等，创建任务队列，将任务处理的goroutine抽象成一个reconcile函数，使用户更方便的编写operator工具。
2. kubebuilder是一个基于controller-runtime框架的命令生成工具。可以用于快速生成和部署crd对象，快速生成controller-runtime框架的基本代码。
3. controller-runtime框架的最核心需要处理的代理及处理reconcile函数，该函数定义了4种错误处理及入队重试的类型。可以根据具体的业务需求选择合适的方法来处理。



参考：

- [controller-runtime源码分析 | 李乾坤的博客](https://qiankunli.github.io/2020/08/10/controller_runtime.html)

- [controller-runtime细节分析 | 李乾坤的博客](https://qiankunli.github.io/2022/11/24/controller_runtime_detail.html)
