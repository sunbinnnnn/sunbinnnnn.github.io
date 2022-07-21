---
title: kubelet
date: 2021-06-23 16:43:01
tags: k8s
typora-root-url: ..
---

在[CSI的文章](https://blog.neilcloud.net/2021/04/20/关于CSI，看这一篇就够了/)中，我们分析了K8S对存储卷的操作，这里除了CSI Controller对PVC的定义外，kubelet也起到了很大的作用，本文主要针对kubelet的实现逻辑进行分析

<!--more-->

## Kubelet的工作原理

kubelet工作在K8S的每个节点上，承担了与节点交互的角色，例如挂载volume卷，创建容器namespace等。

下图是kubelet工作原理的示意图：

<img src="/images/914e097aed10b9ff39b509759f8b1d03.png" alt="img" style="zoom:67%;" />



可以看出，kubelet本身也是一个控制器，遵循k8s控制器的设计模式，



## 代码解析

