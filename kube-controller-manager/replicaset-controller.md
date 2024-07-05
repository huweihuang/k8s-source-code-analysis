---
title: "kube-controller-manager源码分析（四）之 ReplicaSetController"
linkTitle: "ReplicaSetController"
weight: 5
catalog: true
date: 2024-6-9 21:23:24
subtitle:
header-img: "https://res.cloudinary.com/dqxtn0ick/image/upload/v1542285471/header/building.jpg"
tags:
- 源码分析
catagories:
- 源码分析
---

本文主要分析replicaset-controller的源码逻辑，replicas对象创建主要是由deployment-controller中封装。而replicas是pod的维护控制器。可以把replicas理解为deployment中的版本控制器，该控制器封装每次版本的pod对象。

# 1. startReplicaSetController

startReplicaSetController函数是ReplicaSetController的入口函数。基本的操作即new controller对象，然后起一个goroutine运行run函数。

```go
func startReplicaSetController(ctx context.Context, controllerContext ControllerContext) (controller.Interface, bool, error) {
	go replicaset.NewReplicaSetController(
		klog.FromContext(ctx),
		controllerContext.InformerFactory.Apps().V1().ReplicaSets(),
		controllerContext.InformerFactory.Core().V1().Pods(),
		controllerContext.ClientBuilder.ClientOrDie("replicaset-controller"),
		replicaset.BurstReplicas,
	).Run(ctx, int(controllerContext.ComponentConfig.ReplicaSetController.ConcurrentRSSyncs))
	return nil, true, nil
}
```

# 2. NewReplicaSetController

NewReplicaSetController初始化controller对象，最终通过NewBaseController实现具体的初始化操作。

```go
// NewReplicaSetController configures a replica set controller with the specified event recorder
func NewReplicaSetController(logger klog.Logger, rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int) *ReplicaSetController {
	eventBroadcaster := record.NewBroadcaster()
	if err := metrics.Register(legacyregistry.Register); err != nil {
		logger.Error(err, "unable to register metrics")
	}
	return NewBaseController(logger, rsInformer, podInformer, kubeClient, burstReplicas,
		apps.SchemeGroupVersion.WithKind("ReplicaSet"),
		"replicaset_controller",
		"replicaset",
		controller.RealPodControl{
			KubeClient: kubeClient,
			Recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "replicaset-controller"}),
		},
		eventBroadcaster,
	)
}
```

## 2.1. NewBaseController

NewBaseController是一个常见的k8s controller构建函数，主要包括以下几个部分：

- 初始化常用client，包括kube client
- 添加event handler对象。
- 添加informer索引。
- 添加informer syncCache的函数，在处理controller逻辑前先同步一下etcd的数据到本地cache。
- 赋值syncHandler函数，是具体实现controller逻辑的函数。

```go
func NewBaseController(logger klog.Logger, rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int,
	gvk schema.GroupVersionKind, metricOwnerName, queueName string, podControl controller.PodControlInterface, eventBroadcaster record.EventBroadcaster) *ReplicaSetController {

  // 初始化常用配置
	rsc := &ReplicaSetController{
		GroupVersionKind: gvk,
		kubeClient:       kubeClient,
		podControl:       podControl,
		eventBroadcaster: eventBroadcaster,
		burstReplicas:    burstReplicas,
		expectations:     controller.NewUIDTrackingControllerExpectations(controller.NewControllerExpectations()),
		queue:            workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), queueName),
	}

  // 添加event handler对象
	rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			rsc.addRS(logger, obj)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			rsc.updateRS(logger, oldObj, newObj)
		},
		DeleteFunc: func(obj interface{}) {
			rsc.deleteRS(logger, obj)
		},
	})
  
  // 添加informer索引
	rsInformer.Informer().AddIndexers(cache.Indexers{
		controllerUIDIndex: func(obj interface{}) ([]string, error) {
			rs, ok := obj.(*apps.ReplicaSet)
			if !ok {
				return []string{}, nil
			}
			controllerRef := metav1.GetControllerOf(rs)
			if controllerRef == nil {
				return []string{}, nil
			}
			return []string{string(controllerRef.UID)}, nil
		},
	})
	rsc.rsIndexer = rsInformer.Informer().GetIndexer()
	rsc.rsLister = rsInformer.Lister()
  
  // 初始化informer的sync函数
	rsc.rsListerSynced = rsInformer.Informer().HasSynced

	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			rsc.addPod(logger, obj)
		},
		// This invokes the ReplicaSet for every pod change, eg: host assignment. Though this might seem like
		// overkill the most frequent pod update is status, and the associated ReplicaSet will only list from
		// local storage, so it should be ok.
		UpdateFunc: func(oldObj, newObj interface{}) {
			rsc.updatePod(logger, oldObj, newObj)
		},
		DeleteFunc: func(obj interface{}) {
			rsc.deletePod(logger, obj)
		},
	})
	rsc.podLister = podInformer.Lister()
	rsc.podListerSynced = podInformer.Informer().HasSynced

  // 初始化syncHandler函数，该函数为具体实现controller业务逻辑的函数。
	rsc.syncHandler = rsc.syncReplicaSet

	return rsc
}
```

# 3. Run

`Run`函数仍然是k8s controller的代码风格。主要包含了以下几个部分。

- 同步本地cache内容
- 运行多个不退出的goroutine处理控制器逻辑。

```go
func (rsc *ReplicaSetController) Run(ctx context.Context, workers int) {
	// 已删除非核心逻辑代码。
	defer rsc.queue.ShutDown()

  // 处理前同步下cache的内容。
	if !cache.WaitForNamedCacheSync(rsc.Kind, ctx.Done(), rsc.podListerSynced, rsc.rsListerSynced) {
		return
	}

  // 运行指定个数的goroutine来处理controller逻辑。
	for i := 0; i < workers; i++ {
		go wait.UntilWithContext(ctx, rsc.worker, time.Second)
	}

	<-ctx.Done()
}
```

## 3.1. processNextWorkItem

`processNextWorkItemz`主要运行syncHandler函数，和对返回的错误进行处理。

- 如果错误为空，则不再入队。
- 如果错误不为空，则入队重新处理。

```go
func (rsc *ReplicaSetController) worker(ctx context.Context) {
	for rsc.processNextWorkItem(ctx) {
	}
}

func (rsc *ReplicaSetController) processNextWorkItem(ctx context.Context) bool {
	key, quit := rsc.queue.Get()
	if quit {
		return false
	}
	defer rsc.queue.Done(key)

  // 具体的逻辑代码由syncHandler实现。
	err := rsc.syncHandler(ctx, key.(string))
	if err == nil {
    // 如果错误为空，则不再入队
		rsc.queue.Forget(key)
		return true
	}

	utilruntime.HandleError(fmt.Errorf("sync %q failed with %v", key, err))
  // 如果错误不为空，则重新入队
	rsc.queue.AddRateLimited(key)

	return true
}
```

# 4. syncReplicaSet

`syncReplicaSet`是syncHandler的具体实现，常见的syncHandler的实现包含以下几个部分

- 获取集群中的controller对象，例如 rs。
- 获取该controller对象及其子对象的当前状态。
- 对比当前状态与预期状态是否一致。
- 更新当前状态，以上循环直到当前状态达到期望状态。

```go
func (rsc *ReplicaSetController) syncReplicaSet(ctx context.Context, key string) error {
	// 已删除非核心代码
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
  // 获取集群中的rs对象
	rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name)
	if apierrors.IsNotFound(err) {
		logger.V(4).Info("deleted", "kind", rsc.Kind, "key", key)
		rsc.expectations.DeleteExpectations(logger, key)
		return nil
	}

	rsNeedsSync := rsc.expectations.SatisfiedExpectations(logger, key)
  // 获取指定selector下pod
	selector, err := metav1.LabelSelectorAsSelector(rs.Spec.Selector)
	allPods, err := rsc.podLister.Pods(rs.Namespace).List(labels.Everything())
	filteredPods := controller.FilterActivePods(logger, allPods)
	filteredPods, err = rsc.claimPods(ctx, rs, selector, filteredPods)

	// 处理replica逻辑
	var manageReplicasErr error
	if rsNeedsSync && rs.DeletionTimestamp == nil {
		manageReplicasErr = rsc.manageReplicas(ctx, filteredPods, rs)
	}
	rs = rs.DeepCopy()
	newStatus := calculateStatus(rs, filteredPods, manageReplicasErr)

	// 更新状态
	updatedRS, err := updateReplicaSetStatus(logger, rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace), rs, newStatus)

	// Resync the ReplicaSet after MinReadySeconds as a last line of defense to guard against clock-skew.
	if manageReplicasErr == nil && updatedRS.Spec.MinReadySeconds > 0 &&
		updatedRS.Status.ReadyReplicas == *(updatedRS.Spec.Replicas) &&
		updatedRS.Status.AvailableReplicas != *(updatedRS.Spec.Replicas) {
		rsc.queue.AddAfter(key, time.Duration(updatedRS.Spec.MinReadySeconds)*time.Second)
	}
	return manageReplicasErr
}
```

# 5. manageReplicas

`manageReplicas`主要实现pod的创建和删除，从而保证当前rs下的pod跟预期的一致。

```go
func (rsc *ReplicaSetController) manageReplicas(ctx context.Context, filteredPods []*v1.Pod, rs *apps.ReplicaSet) error {
  // 计算当前的pod数量和预期的pod是否一致。
	diff := len(filteredPods) - int(*(rs.Spec.Replicas))
	rsKey, err := controller.KeyFunc(rs)
	// 如果少于预期
	if diff < 0 {
		// 则批量创建pod
		successfulCreations, err := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() error {
			err := rsc.podControl.CreatePods(ctx, rs.Namespace, &rs.Spec.Template, rs, metav1.NewControllerRef(rs, rsc.GroupVersionKind))
		})
	// 如果pod数量多于预期
	} else if diff > 0 {

		relatedPods, err := rsc.getIndirectlyRelatedPods(logger, rs)
		utilruntime.HandleError(err)

		// Choose which Pods to delete, preferring those in earlier phases of startup.
		podsToDelete := getPodsToDelete(filteredPods, relatedPods, diff)


		errCh := make(chan error, diff)
		var wg sync.WaitGroup
		wg.Add(diff)
		for _, pod := range podsToDelete {
			go func(targetPod *v1.Pod) {
				defer wg.Done()
        // 批量删除pod
				if err := rsc.podControl.DeletePod(ctx, rs.Namespace, targetPod.Name, rs); err != nil {
				}
			}(pod)
		}
		wg.Wait()

    // 处理错误
		select {
		case err := <-errCh:
			// all errors have been reported before and they're likely to be the same, so we'll only return the first one we hit.
			if err != nil {
				return err
			}
		default:
		}
	}

	return nil
}
```

# 总结

replicaset-controller的代码逻辑相对简单，基本的代码风格是k8s控制器通用的代码逻辑，由于k8s的代码风格高度一致，因此如果读清楚一类controller的控制逻辑。其他的控制器的代码逻辑大同小异。

参考：

- https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go
