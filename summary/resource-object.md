---
title: "k8s核心数据结构分析"
linkTitle: "k8s核心数据结构分析"
weight: 1
catalog: true
date: 2023-3-10 20:23:24
tags:
- 源码分析
catagories:
- 源码分析
---

> 本文基于《Kubernetes源码剖析》整理，结合k8s v1.22.0代码分析

# 概述

k8s声明式API的思想，以资源描述对象为中心，声明对象的`spec`，通过系统维持`status`状态始终是用户声明的资源描述`spec`，具体可以参考[理解k8s资源对象](https://www.huweihuang.com/kubernetes-notes/concepts/object/understanding-kubernetes-objects.html)。k8s是一个完全以资源为中心的系统，本质是一个资源控制系统--注册、管理、调度资源并维护资源的状态。k8s将资源进行再次分组和版本化，形成Group(资源组)、Version(资源版本)、Resource(资源)【其中kind表示资源种类】

# 资源描述对象

资源描述对象必须的声明元素如下：

- apiVersion：kubernetes API的版本，包含`<group>/<version>`
- kind：kubernetes对象的类型
- metadata：唯一标识该对象的元数据，包括name，UID，可选的namespace
- spec：标识对象的详细信息，不同对象的spec的格式不同，可以嵌套其他对象的字段。

说明：

1. `<group>/<version>/<resource>`表示唯一一种资源对象，例如`apps/v1/deployment`。

2. 每种资源对象都有增删改查的操作方法。具体可包含8类操作接口：
- 增：create

- 删：delete，deletecollection

- 改：update，patch

- 查：get，list，watch
3. 除了内置的k8s资源对象，可用CRD自定义资源对象，在k8s系统中使用。

示例：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

# 资源对象分类及代码目录

# 资源对象通用结构
