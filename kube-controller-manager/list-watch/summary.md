
# 8. 总结

本文分析的部分主要是k8s的`informer`机制，即`List-Watch`机制。

## 8.1. Reflector

`Reflector`的主要作用是watch指定的k8s资源，并将变化同步到本地是`store`中。`Reflector`只会放置指定的`expectedType`类型的资源到`store`中，除非`expectedType`为nil。如果`resyncPeriod`不为零，那么`Reflector`为以`resyncPeriod`为周期定期执行list的操作，这样就可以使用`Reflector`来定期处理所有的对象，也可以逐步处理变化的对象。

## 8.2. ListAndWatch

`ListAndWatch`第一次会列出所有的对象，并获取资源对象的版本号，然后watch资源对象的版本号来查看是否有被变更。首先会将资源版本号设置为0，`list()`可能会导致本地的缓存相对于etcd里面的内容存在延迟，`Reflector`会通过`watch`的方法将延迟的部分补充上，使得本地的缓存数据与etcd的数据保持一致。

## 8.3. DeltaFIFO

`DeltaFIFO`是一个生产者与消费者的队列，其中Reflector是生产者，消费者调用Pop()的方法。

DeltaFIFO主要用在以下场景：

- 希望对象变更最多处理一次
- 处理对象时，希望查看自上次处理对象以来发生的所有事情
- 要处理对象的删除
- 希望定期重新处理对象

## 8.4. store

`Store`是一个通用的存储接口，Reflector通过watch server的方式更新数据到store中，store给Reflector提供本地的缓存，让Reflector可以像消息队列一样的工作。

`Store`实现的是一种可以准确的写入对象和获取对象的机制。

## 8.5. processor

`processor`的主要功能就是记录了所有的回调函数实例(即 ResourceEventHandler 实例)，并负责触发这些函数。在sharedIndexInformer.Run部分会调用processor.run。

流程：

1. listenser的add函数负责将notify装进pendingNotifications。
2. pop函数取出pendingNotifications的第一个nofify,输出到nextCh channel。
3. run函数则负责取出notify，然后根据notify的类型(增加、删除、更新)触发相应的处理函数，这些函数是在不同的`NewXxxcontroller`实现中注册的。

`processor`记录了不同类似的事件函数，其中事件函数在`NewXxxController`构造函数部分注册，具体事件函数的处理，一般是将需要处理的对象加入对应的controller的任务队列中，然后由类似`syncDeployment`的同步函数来维持期望状态的同步逻辑。

## 8.6. 主要步骤

1. 在controller-manager的Run函数部分调用了InformerFactory.Start的方法，Start方法初始化各种类型的informer，并且每个类型起了个informer.Run的goroutine。
2. informer.Run的部分先生成一个DeltaFIFO的队列来存储对象变化的数据。然后调用processor.Run和controller.Run函数。
3. controller.Run函数会生成一个Reflector，`Reflector`的主要作用是watch指定的k8s资源，并将变化同步到本地是`store`中。`Reflector`以`resyncPeriod`为周期定期执行list的操作，这样就可以使用`Reflector`来定期处理所有的对象，也可以逐步处理变化的对象。
4. Reflector接着执行ListAndWatch函数，ListAndWatch第一次会列出所有的对象，并获取资源对象的版本号，然后watch资源对象的版本号来查看是否有被变更。首先会将资源版本号设置为0，`list()`可能会导致本地的缓存相对于etcd里面的内容存在延迟，`Reflector`会通过`watch`的方法将延迟的部分补充上，使得本地的缓存数据与etcd的数据保持一致。
5. controller.Run函数还会调用processLoop函数，processLoop通过调用HandleDeltas，再调用distribute，processorListener.add最终将不同更新类型的对象加入`processorListener`的channel中，供processorListener.Run使用。
6. processor的主要功能就是记录了所有的回调函数实例(即 ResourceEventHandler 实例)，并负责触发这些函数。processor记录了不同类型的事件函数，其中事件函数在NewXxxController构造函数部分注册，具体事件函数的处理，一般是将需要处理的对象加入对应的controller的任务队列中，然后由类似syncDeployment的同步函数来维持期望状态的同步逻辑。



参考文章：

- <https://github.com/kubernetes/client-go/tree/master/tools/cache>

- <https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md>
- <https://github.com/kubernetes/client-go/blob/master/examples/workqueue/main.go>
