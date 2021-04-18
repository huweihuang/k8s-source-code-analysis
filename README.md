# Kubernetes 源码分析笔记

> 本系列是 [Kubernetes 源码分析笔记](https://www.huweihuang.com/k8s-source-code-analysis/)
> 
> 更多的学习笔记请参考：
> - [Kubernetes 学习笔记](https://www.huweihuang.com/kubernetes-notes/)
> - [Golang 学习笔记](https://www.huweihuang.com/golang-notes/)
> - [Linux 学习笔记](https://www.huweihuang.com/linux-notes/)
> - [数据结构学习笔记](https://www.huweihuang.com/data-structure-notes/)
>
> 个人博客：[www.huweihuang.com](https://www.huweihuang.com/)


# 目录

## 前言

* [序言](README.md)

## kube-apiserver

* [NewAPIServerCommand](kube-apiserver/NewAPIServerCommand.md)


## kube-controller-manager

* [源码思维导图](kube-controller-manager/controller-manager-xmind.md)
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

* [源码思维导图](kube-scheduler/scheduler-xmind.md)
 * [NewSchedulerCommand](kube-scheduler/NewSchedulerCommand.md)
 * [registerAlgorithmProvider](kube-scheduler/registerAlgorithmProvider.md)
 * [scheduleOne](kube-scheduler/scheduleOne.md)
 * [findNodesThatFit](kube-scheduler/findNodesThatFit.md)
 * [PrioritizeNodes](kube-scheduler/PrioritizeNodes.md)
 * [preempt](kube-scheduler/preempt.md)

## kubelet

* [源码思维导图](kubelet/kubelet-xmind.md)
* [NewKubeletCommand](kubelet/NewKubeletCommand.md)
* [NewMainKubelet](kubelet/NewMainKubelet.md)
* [startKubelet](kubelet/startKubelet.md)
* [syncLoopIteration](kubelet/syncLoopIteration.md)
* [syncPod](kubelet/syncPod.md)
  
# 赞赏

> 如果觉得文章有帮助的话，可以打赏一下，谢谢！

<img src="https://res.cloudinary.com/dqxtn0ick/image/upload/v1551599963/blog/donate.jpg" width="70%"/>
