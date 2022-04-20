---
title: openstack-helm离线部署
date: 2022-04-19 14:27:23
tags:
---



| 节点名 | 角色           |
| ------ | -------------- |
| knode1 | 融合节点       |
| knode2 | 融合节点       |
| knode3 | 融合节点       |
| knode4 | 计算、存储节点 |



# 一.helm准备

1.将helm二进制复制到`/usr/bin`下

```
cp helm /usr/bin/helm
chmod +x /usr/bin/helm
```

2.去除master节点的污点

```
kubectl taint nodes knode1 node-role.kubernetes.io/master:NoSchedule-
kubectl taint nodes knode2 node-role.kubernetes.io/master:NoSchedule-
kubectl taint nodes knode3 node-role.kubernetes.io/master:NoSchedule-
```

3.为节点添加label

```
kubectl label nodes knode1 ceph-control-plane=enabled openstack-control-plane=enabled
kubectl label nodes knode2 ceph-control-plane=enabled openstack-control-plane=enabled
kubectl label nodes knode3 ceph-control-plane=enabled openstack-control-plane=enabled
kubectl label nodes knode4 openstack-compute-node=enabled
```



# 二.安装rook-ceph

1.安装rook-ceph-operator

```
helm install --create-namespace --namespace rook-ceph rook-ceph ./rook-ceph-0.0.1.tgz  --set image.tag=v1.8.8
```

2.创建ceph cluster、storageClass

```
helm install --create-namespace --namespace rook-ceph rook-ceph-cluster ./rook-ceph-cluster-0.0.1.tgz --set cephClusterSpec.dashboard.enabled=false
```

> 删除时，需要先执行```kubectl -n rook-ceph patch cephcluster rook-ceph --type merge -p '{"spec":{"cleanupPolicy":{"confirmation":"yes-really-destroy-data"}}}'```
>

3.等待集群创建完成

```
 kubectl  wait --for=condition=Ready -n rook-ceph cephcluster rook-ceph
```



# 三.安装Registry、ChartMuseum

