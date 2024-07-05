---
title: "kube-controller-manager源码分析（五）之 DaemonSetController"
linkTitle: "DaemonSetController"
weight: 6
catalog: true
date: 2024-6-10 21:23:24
subtitle:
header-img: "https://res.cloudinary.com/dqxtn0ick/image/upload/v1542285471/header/building.jpg"
tags:
- 源码分析
catagories:
- 源码分析
---

本文主要分析`DaemonSetController`的源码逻辑，daemonset是运行在指定节点上的服务，常用来作为agent类的服务来配置，也是k8s最常用的控制器之一。

# 1. startDaemonSetController

startDaemonSetController是入口函数，先New后Run。

```go
func startDaemonSetController(ctx context.Context, controllerContext ControllerContext) (controller.Interface, bool, error) {
   dsc, err := daemon.NewDaemonSetsController(
      ctx,
      controllerContext.InformerFactory.Apps().V1().DaemonSets(),
      controllerContext.InformerFactory.Apps().V1().ControllerRevisions(),
      controllerContext.InformerFactory.Core().V1().Pods(),
      controllerContext.InformerFactory.Core().V1().Nodes(),
      controllerContext.ClientBuilder.ClientOrDie("daemon-set-controller"),
      flowcontrol.NewBackOff(1*time.Second, 15*time.Minute),
   )
   if err != nil {
      return nil, true, fmt.Errorf("error creating DaemonSets controller: %v", err)
   }
   go dsc.Run(ctx, int(controllerContext.ComponentConfig.DaemonSetController.ConcurrentDaemonSetSyncs))
   return nil, true, nil
}
```

# 2. NewDaemonSetsController

NewDaemonSetsController仍然是常见的k8s controller初始化逻辑:

- 常用配置初始化
- 根据所需要的对象，添加event handler，便于监听所需要对象的事件变化，此处包括ds, pod，node三个对象。
- 赋值syncHandler，即具体实现是syncDaemonSet函数。

```go
func NewDaemonSetsController(
	ctx context.Context,
	daemonSetInformer appsinformers.DaemonSetInformer,
	historyInformer appsinformers.ControllerRevisionInformer,
	podInformer coreinformers.PodInformer,
	nodeInformer coreinformers.NodeInformer,
	kubeClient clientset.Interface,
	failedPodsBackoff *flowcontrol.Backoff,
) (*DaemonSetsController, error) {

  // 常用配置初始化
	dsc := &DaemonSetsController{
		kubeClient:       kubeClient,
		eventBroadcaster: eventBroadcaster,
		eventRecorder:    eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "daemonset-controller"}),
		podControl: controller.RealPodControl{
			KubeClient: kubeClient,
			Recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "daemonset-controller"}),
		},
		crControl: controller.RealControllerRevisionControl{
			KubeClient: kubeClient,
		},
		burstReplicas: BurstReplicas,
		expectations:  controller.NewControllerExpectations(),
		queue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "daemonset"),
	}

  // 添加event handler
	daemonSetInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			dsc.addDaemonset(logger, obj)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			dsc.updateDaemonset(logger, oldObj, newObj)
		},
		DeleteFunc: func(obj interface{}) {
			dsc.deleteDaemonset(logger, obj)
		},
	})
	dsc.dsLister = daemonSetInformer.Lister()
	dsc.dsStoreSynced = daemonSetInformer.Informer().HasSynced

	historyInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			dsc.addHistory(logger, obj)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			dsc.updateHistory(logger, oldObj, newObj)
		},
		DeleteFunc: func(obj interface{}) {
			dsc.deleteHistory(logger, obj)
		},
	})
	dsc.historyLister = historyInformer.Lister()
	dsc.historyStoreSynced = historyInformer.Informer().HasSynced

	// 添加pod的event handler
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			dsc.addPod(logger, obj)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			dsc.updatePod(logger, oldObj, newObj)
		},
		DeleteFunc: func(obj interface{}) {
			dsc.deletePod(logger, obj)
		},
	})
	dsc.podLister = podInformer.Lister()
	dsc.podStoreSynced = podInformer.Informer().HasSynced

  // 添加node的event handler
	nodeInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			dsc.addNode(logger, obj)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			dsc.updateNode(logger, oldObj, newObj)
		},
	},
	)
	dsc.nodeStoreSynced = nodeInformer.Informer().HasSynced
	dsc.nodeLister = nodeInformer.Lister()

  // 赋值syncHandler，具体的controller处理代码
	dsc.syncHandler = dsc.syncDaemonSet
	dsc.enqueueDaemonSet = dsc.enqueue

	dsc.failedPodsBackoff = failedPodsBackoff

	return dsc, nil
}
```

# 3. Run

`Run`函数与其他controller的逻辑一致不再分析，具体可以阅读本源码分析系列的replicaset-controller分析。

```go
func (dsc *DaemonSetsController) Run(ctx context.Context, workers int) {
	defer utilruntime.HandleCrash()

	dsc.eventBroadcaster.StartStructuredLogging(0)
	dsc.eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: dsc.kubeClient.CoreV1().Events("")})
	defer dsc.eventBroadcaster.Shutdown()

	defer dsc.queue.ShutDown()

	logger := klog.FromContext(ctx)
	logger.Info("Starting daemon sets controller")
	defer logger.Info("Shutting down daemon sets controller")

	if !cache.WaitForNamedCacheSync("daemon sets", ctx.Done(), dsc.podStoreSynced, dsc.nodeStoreSynced, dsc.historyStoreSynced, dsc.dsStoreSynced) {
		return
	}

	for i := 0; i < workers; i++ {
		go wait.UntilWithContext(ctx, dsc.runWorker, time.Second)
	}

	go wait.Until(dsc.failedPodsBackoff.GC, BackoffGCInterval, ctx.Done())

	<-ctx.Done()
}
```

## 3.1. processNextWorkItem

`processNextWorkItem`可参考replicaset-controller对该部分的分析。

```go
func (dsc *DaemonSetsController) runWorker(ctx context.Context) {
   for dsc.processNextWorkItem(ctx) {
   }
}

// processNextWorkItem deals with one key off the queue.  It returns false when it's time to quit.
func (dsc *DaemonSetsController) processNextWorkItem(ctx context.Context) bool {
   dsKey, quit := dsc.queue.Get()
   if quit {
      return false
   }
   defer dsc.queue.Done(dsKey)

   err := dsc.syncHandler(ctx, dsKey.(string))
   if err == nil {
      dsc.queue.Forget(dsKey)
      return true
   }

   utilruntime.HandleError(fmt.Errorf("%v failed with : %v", dsKey, err))
   dsc.queue.AddRateLimited(dsKey)

   return true
}
```

# 4. syncDaemonSet

`syncDaemonSet`是控制器具体的实现逻辑，即每个控制器的核心实现大部分在syncHandler这个函数中。

```go
func (dsc *DaemonSetsController) syncDaemonSet(ctx context.Context, key string) error {
   // 为了突出代码重点，已删除非必要部分。
	 // 获取集群中的ds对象
   namespace, name, err := cache.SplitMetaNamespaceKey(key)
   ds, err := dsc.dsLister.DaemonSets(namespace).Get(name)

	 // 获取机器列表
   nodeList, err := dsc.nodeLister.List(labels.Everything())
   everything := metav1.LabelSelector{}
   if reflect.DeepEqual(ds.Spec.Selector, &everything) {
      dsc.eventRecorder.Eventf(ds, v1.EventTypeWarning, SelectingAllReason, "This daemon set is selecting all pods. A non-empty selector is required.")
      return nil
   }

   // Construct histories of the DaemonSet, and get the hash of current history
   cur, old, err := dsc.constructHistory(ctx, ds)
   hash := cur.Labels[apps.DefaultDaemonSetUniqueLabelKey]

   if !dsc.expectations.SatisfiedExpectations(logger, dsKey) {
      // Only update status. Don't raise observedGeneration since controller didn't process object of that generation.
      return dsc.updateDaemonSetStatus(ctx, ds, nodeList, hash, false)
   }

   // 处理daemonset中的pod创建及删除
   err = dsc.updateDaemonSet(ctx, ds, nodeList, hash, dsKey, old)
   // 更新状态
   statusErr := dsc.updateDaemonSetStatus(ctx, ds, nodeList, hash, true)
   switch {
   case err != nil && statusErr != nil:
      logger.Error(statusErr, "Failed to update status", "daemonSet", klog.KObj(ds))
      return err
   case err != nil:
      return err
   case statusErr != nil:
      return statusErr
   }

   return nil
}
```

# 5. manage

`manage`是具体的daemonset的pod创建及删除的具体代码。manage入口在updateDaemonSet的函数中。

```go
func (dsc *DaemonSetsController) updateDaemonSet(ctx context.Context, ds *apps.DaemonSet, nodeList []*v1.Node, hash, key string, old []*apps.ControllerRevision) error {
  // manage处理pod的创建及删除
	err := dsc.manage(ctx, ds, nodeList, hash)

	err = dsc.cleanupHistory(ctx, ds, old)
	if err != nil {
		return fmt.Errorf("failed to clean up revisions of DaemonSet: %w", err)
	}

	return nil
}
```

以下是manage的具体代码

```go
func (dsc *DaemonSetsController) manage(ctx context.Context, ds *apps.DaemonSet, nodeList []*v1.Node, hash string) error {
   // 查找daemonset中pod和node的映射
   nodeToDaemonPods, err := dsc.getNodesToDaemonPods(ctx, ds, false)
	 // 计算需要创建和删除的pod
   var nodesNeedingDaemonPods, podsToDelete []string
   for _, node := range nodeList {
      nodesNeedingDaemonPodsOnNode, podsToDeleteOnNode := dsc.podsShouldBeOnNode(
         logger, node, nodeToDaemonPods, ds, hash)

      nodesNeedingDaemonPods = append(nodesNeedingDaemonPods, nodesNeedingDaemonPodsOnNode...)
      podsToDelete = append(podsToDelete, podsToDeleteOnNode...)
   }

   // Remove unscheduled pods assigned to not existing nodes when daemonset pods are scheduled by scheduler.
   // If node doesn't exist then pods are never scheduled and can't be deleted by PodGCController.
   podsToDelete = append(podsToDelete, getUnscheduledPodsWithoutNode(nodeList, nodeToDaemonPods)...)

   // 根据上述的计算结果，实现具体创建和删除pod的操作
   if err = dsc.syncNodes(ctx, ds, podsToDelete, nodesNeedingDaemonPods, hash); err != nil {
      return err
   }

   return nil
}
```

# 6. syncNodes

`syncNodes`根据传入的需要创建和删除的pod，实现具体的创建和删除pod的操作。

```go
func (dsc *DaemonSetsController) syncNodes(ctx context.Context, ds *apps.DaemonSet, podsToDelete, nodesNeedingDaemonPods []string, hash string) error {
	// 已删除次要代码
	dsKey, err := controller.KeyFunc(ds)

	batchSize := integer.IntMin(createDiff, controller.SlowStartInitialBatchSize)
	for pos := 0; createDiff > pos; batchSize, pos = integer.IntMin(2*batchSize, createDiff-(pos+batchSize)), pos+batchSize {
		errorCount := len(errCh)
		createWait.Add(batchSize)
		for i := pos; i < pos+batchSize; i++ {
			go func(ix int) {
				defer createWait.Done()
				// 批量创建pod
				err := dsc.podControl.CreatePods(ctx, ds.Namespace, podTemplate,
					ds, metav1.NewControllerRef(ds, controllerKind))
			}(i)
		}
		createWait.Wait()
	}

	deleteWait := sync.WaitGroup{}
	deleteWait.Add(deleteDiff)
	for i := 0; i < deleteDiff; i++ {
		go func(ix int) {
			defer deleteWait.Done()
      // 批量删除pod
			if err := dsc.podControl.DeletePod(ctx, ds.Namespace, podsToDelete[ix], ds); err != nil {
				dsc.expectations.DeletionObserved(logger, dsKey)
			}
		}(i)
	}
	deleteWait.Wait()

	// 处理错误
	errors := []error{}
	close(errCh)
	for err := range errCh {
		errors = append(errors, err)
	}
	return utilerrors.NewAggregate(errors)
}
```

参考：

- https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/daemon/daemon_controller.go
