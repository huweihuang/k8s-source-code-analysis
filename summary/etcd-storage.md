---
title: "k8s中Etcd存储的实现"
linkTitle: "k8s中Etcd存储的实现"
weight: 2
catalog: true
date: 2023-3-10 20:23:24
tags:
- 源码分析
catagories:
- 源码分析
---

> 本文基于《Kubernetes源码剖析》整理，结合k8s v1.22.0代码分析

# 概述

k8s基于Etcd作为存储，Etcd是分布式的KV存储集群，Etcd中存储了k8s的`元数据`、`事件数据`、`状态数据`等，数据前缀为`/registry`下，具体的各类对象的key可以参考[Etcd中的k8s数据](https://www.huweihuang.com/kubernetes-notes/etcd/k8s-etcd-data.html)。

> Etcd作为k8s唯一存储，兼具了`MySQL存储元数据`和`消息队列存储任务事件`的功能。

# Etcd存储架构设计

k8s对etcd的操作进行了分层封装，从上(k8s)到下(etcd)，分别为：

- RESTStorage（k8s对象操作封装）
- RegistryStore（BeforeFunc,AfterFunc）
- Storage.Interface（增删改查方法封装）
- CacherStorage（Etcd缓存层）
- UnderlyingStorage（与Etcd交互）

# RESTStorage存储接口

每种k8s资源实现RESTStorage接口，接口代码如下：

> kubernetes/staging/src/k8s.io/apiserver/pkg/registry/rest/rest.go 

```go
// Storage is a generic interface for RESTful storage services.
// Resources which are exported to the RESTful API of apiserver need to implement this interface. It is expected
// that objects may implement any of the below interfaces.
type Storage interface {
    // New returns an empty object that can be used with Create and Update after request data has been put into it.
    // This object must be a pointer type for use with Codec.DecodeInto([]byte, runtime.Object)
    New() runtime.Object
}
```

k8s基于etcd相关封装代码主要在`/pkg/registry`目录下。

其中每种资源对于storage接口的实现定义在`/pkg/registry/<资源组>/<资源>/storage/storage.go`。以下以`deployment`为例串联k8s中关于etcd的调用流程，调用顺序从上(k8s)到下(etcd)。

## DeploymentStorage

> kubernetes/pkg/registry/apps/deployment/storage/storage.go  

```go
// DeploymentStorage includes dummy storage for Deployments and for Scale subresource.
type DeploymentStorage struct {
    Deployment *REST
    Status     *StatusREST
    Scale      *ScaleREST
    Rollback   *RollbackREST
}

// REST implements a RESTStorage for Deployments.
type REST struct {
    *genericregistry.Store
    categories []string
}

// StatusREST implements the REST endpoint for changing the status of a deployment
type StatusREST struct {
    store *genericregistry.Store
}

// ScaleREST implements a Scale for Deployment.
type ScaleREST struct {
    store *genericregistry.Store
}

// RollbackREST implements the REST endpoint for initiating the rollback of a deployment
type RollbackREST struct {
    store *genericregistry.Store
}
```
