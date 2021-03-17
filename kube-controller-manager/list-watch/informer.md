# kube-controller-manager源码分析（三）之 Informer机制

> 以下代码分析基于 `kubernetes v1.12.0` 版本。

# list-watch机制

本文主要分析k8s中各个核心组件经常使用到的`Informer`机制(即List-Watch)。该部分的代码主要位于`client-go`这个第三方包中。

此部分的逻辑主要位于`/vendor/k8s.io/client-go/tools/cache`包中，代码目录结构如下：

```bash
cache
├── controller.go  # 包含：Config、Run、processLoop、NewInformer、NewIndexerInformer
├── delta_fifo.go  # 包含：NewDeltaFIFO、DeltaFIFO、AddIfNotPresent
├── expiration_cache.go
├── expiration_cache_fakes.go
├── fake_custom_store.go
├── fifo.go   # 包含：Queue、FIFO、NewFIFO
├── heap.go
├── index.go    # 包含：Indexer、MetaNamespaceIndexFunc
├── listers.go
├── listwatch.go   # 包含：ListerWatcher、ListWatch、List、Watch
├── mutation_cache.go
├── mutation_detector.go
├── reflector.go   # 包含：Reflector、NewReflector、Run、ListAndWatch
├── reflector_metrics.go
├── shared_informer.go  # 包含：NewSharedInformer、WaitForCacheSync、Run、HasSynced
├── store.go  # 包含：Store、MetaNamespaceKeyFunc、SplitMetaNamespaceKey
├── testing
│   ├── fake_controller_source.go
├── thread_safe_store.go  # 包含：ThreadSafeStore、threadSafeMap
├── undelta_store.go
```

# 1. 原理示意图

[示意图1](https://www.kubernetes.org.cn/1283.html)：

<img src="https://res.cloudinary.com/dqxtn0ick/image/upload/v1555472372/article/code-analysis/informer/client-go.png" width="100%">

[示意图2](https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md)：

<img src="https://res.cloudinary.com/dqxtn0ick/image/upload/v1555479782/article/code-analysis/informer/client-go-controller-interaction.jpg" width="100%">

# 2. 组件

## 2.1. client-go组件

- `Reflector`：reflector用来watch特定的k8s API资源。具体的实现是通过`ListAndWatch`的方法，watch可以是k8s内建的资源或者是自定义的资源。当reflector通过watch API接收到有关新资源实例存在的通知时，它使用相应的列表API获取新创建的对象，并将其放入watchHandler函数内的Delta Fifo队列中。

- `Informer`：informer从Delta Fifo队列中弹出对象。执行此操作的功能是processLoop。base controller的作用是保存对象以供以后检索，并调用我们的控制器将对象传递给它。

- `Indexer`：索引器提供对象的索引功能。典型的索引用例是基于对象标签创建索引。 Indexer可以根据多个索引函数维护索引。Indexer使用线程安全的数据存储来存储对象及其键。 在Store中定义了一个名为`MetaNamespaceKeyFunc`的默认函数，该函数生成对象的键作为该对象的`<namespace> / <name>`组合。

## 2.2. 自定义controller组件

- `Informer reference`：指的是Informer实例的引用，定义如何使用自定义资源对象。 自定义控制器代码需要创建对应的Informer。

- `Indexer reference`: 自定义控制器对Indexer实例的引用。自定义控制器需要创建对应的Indexser。

> client-go中提供`NewIndexerInformer`函数可以创建Informer 和 Indexer。

- `Resource Event Handlers`：资源事件回调函数，当它想要将对象传递给控制器时，它将被调用。 编写这些函数的典型模式是获取调度对象的key，并将该key排入工作队列以进行进一步处理。

- `Work queue`：任务队列。 编写资源事件处理程序函数以提取传递的对象的key并将其添加到任务队列。

- `Process Item`：处理任务队列中对象的函数， 这些函数通常使用Indexer引用或Listing包装器来重试与该key对应的对象。
