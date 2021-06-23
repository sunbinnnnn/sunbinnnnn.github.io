---
title: garbage-collector-controller
date: 2021-06-15 10:44:16
tags: k8s
---

Kubernetes在删除对象时，其对应的controller并不会真正去删除对象，删除对象工作是由GarbageCollectorController负责的。

<!--more-->

当删除一个对象时，会根据删除策略来对资源进行回收处理。

### K8S中的删除策略

Orphan

Foreground

Background

