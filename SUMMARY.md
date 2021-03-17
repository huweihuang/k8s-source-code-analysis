# Summary

## 前言

* [序言](README.md)

## kube-apiserver

* [NewAPIServerCommand](kube-apiserver/NewAPIServerCommand.md)


## kube-controller-manager

* [NewControllerManagerCommand](kube-controller-manager/NewControllerManagerCommand.md)
* [DeploymentController](kube-controller-manager/deployment-controller.md)
* [Informer机制]()
    * [Informer原理](kube-controller-manager/list-watch/informer.md)
    * [sharedIndexInformer](kube-controller-manager/list-watch/sharedIndexInformer.md)
    * [Reflector](kube-controller-manager/list-watch/reflector.md)    
    * [DeltaFIFO](kube-controller-manager/list-watch/DeltaFIFO.md)
    * [processLoop](kube-controller-manager/list-watch/processLoop.md)
    * [总结](kube-controller-manager/list-watch/summary.md)

## kube-scheduler

 * [NewSchedulerCommand](kube-scheduler/NewSchedulerCommand.md)
 * [registerAlgorithmProvider](kube-scheduler/registerAlgorithmProvider.md)
 * [scheduleOne](kube-scheduler/scheduleOne.md)
 * [findNodesThatFit](kube-scheduler/findNodesThatFit.md)
 * [PrioritizeNodes](kube-scheduler/PrioritizeNodes.md)
 * [preempt](kube-scheduler/preempt.md)

## kubelet

* [NewKubeletCommand](kubelet/NewKubeletCommand.md)
* [NewMainKubelet](kubelet/NewMainKubelet.md)
* [startKubelet](kubelet/startKubelet.md)
* [syncLoopIteration](kubelet/syncLoopIteration.md)
* [syncPod](kubelet/syncPod.md)
