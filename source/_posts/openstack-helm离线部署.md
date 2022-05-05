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

> 文章中引用的chart包是经过改造的，如需要，请联系笔者
>

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
helm install --create-namespace --namespace rook-ceph rook-ceph ./charts/rook-ceph-0.0.1.tgz  --set image.tag=v1.8.8
```

2.创建ceph cluster、storageClass

```
helm install --create-namespace --namespace rook-ceph rook-ceph-cluster ./charts/rook-ceph-cluster-0.0.1.tgz --set cephClusterSpec.dashboard.enabled=false
```

> 删除时，需要先执行```kubectl -n rook-ceph patch cephcluster rook-ceph --type merge -p '{"spec":{"cleanupPolicy":{"confirmation":"yes-really-destroy-data"}}}'```
>

3.等待集群创建完成

```
 kubectl  wait --for=condition=Ready -n rook-ceph cephcluster rook-ceph --timeout=500s
```



# 三.安装Registry、ChartMuseum

1.安装并配置docker-registry

```
helm install docker-registry --create-namespace --namespace registry ./charts/docker-registry-2.1.0.tgz --set persistence.size=60Gi
```

2.设置私有仓库和设置dns地址（**需要在所有节点执行**）

```
# replace nameserver to kubedns, every nodes should do below
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "insecure-registries" : ["docker-registry.registry.svc.cluster.local"]
}
EOF
systemctl daemon-reload
systemctl restart docker

NAMESERVER=kubectl get svc -n kube-system|awk '{ print $3 }'|grep -v IP
echo "nameserver $NAMESERVER" > /etc/resolv.conf
```

> 测试：
>
> docker tag rook/ceph:v1.8.8 docker-registry.registry.svc.cluster.local/ceph:v1.8.8
>
> docker push docker-registry.registry.svc.cluster.local/ceph:v1.8.8

3.导入镜像到registry

```
# 修改Kubelet ImageGC配置，防止kubelet删除节点镜像
echo "imageGCHighThresholdPercent: 100" >> /var/lib/kubelet/config.yaml
systemctl restart kubelet

docker load -q < images/components/all_components_images.tar.gz
docker images | grep docker-registry.registry.svc.cluster.local | awk '{print "docker push -q "$1":"$2}'|sh

sed -i '/imageGCHighThresholdPercent/d'  /var/lib/kubelet/config.yaml
systemctl restart kubelet

```

4.安装并配置chartmuseum

```
helm install chartmuseum --create-namespace --namespace registry ./charts/chartmuseum-3.7.0.tgz
helm repo add chartmuseum http://chartmuseum.registry.svc.cluster.local:8080/
helm repo update
```

5.安装并配置openstackclient

```
sudo -H -E pip3 install  --no-index --find-links=./pip3_pack/ cmd2 python-openstackclient python-heatclient --ignore-installed

sudo -H mkdir -p /etc/openstack
sudo -H chown -R $(id -un): /etc/openstack
FEATURE_GATE="tls"; if [[ ${FEATURE_GATES//,/ } =~ (^|[[:space:]])${FEATURE_GATE}($|[[:space:]]) ]]; then
  tee /etc/openstack/clouds.yaml << EOF
  clouds:
    openstack_helm:
      region_name: RegionOne
      identity_api_version: 3
      cacert: /etc/openstack-helm/certs/ca/ca.pem
      auth:
        username: 'admin'
        password: 'password'
        project_name: 'admin'
        project_domain_name: 'default'
        user_domain_name: 'default'
        auth_url: 'https://keystone.openstack.svc.cluster.local/v3'
EOF
else
  tee /etc/openstack/clouds.yaml << EOF
  clouds:
    openstack_helm:
      region_name: RegionOne
      identity_api_version: 3
      auth:
        username: 'admin'
        password: 'password'
        project_name: 'admin'
        project_domain_name: 'default'
        user_domain_name: 'default'
        auth_url: 'http://keystone.openstack.svc.cluster.local/v3'
EOF
fi
```

6.导入charts

```
for i in `ls ./charts`;do curl --data-binary "@./charts/$i" http://chartmuseum.registry.svc.cluster.local:8080/api/charts;done
helm repo update
```



# 三.部署、配置平台中间件

1.安装ingress-controller

```
helm install nginx-ingress-controller --namespace kube-system chartmuseum/nginx-ingress

# 等待pod启动
kubectl  wait --for=condition=Available -n kube-system deployment/nginx-ingress-controller-nginx-ingress --timeout=300s
```

2.部署mariadb

```
helm install mariadb --namespace openstack chartmuseum/mariadb --set volume.enabled=true --set volume.class_name=ceph-block --set volume.size=10Gi

# 等待3个pod启动
kubectl  wait --for=condition=Ready -n openstack pods/mariadb-server-2
kubectl  wait --for=condition=Ready -n openstack pods/mariadb-server-1
kubectl  wait --for=condition=Ready -n openstack pods/mariadb-server-0
```

> Pod一直无法启动？
>
> 一般情况下是mariaDB集群时间采样与实际时间不符导致，需要检查以下两点：
>
> 1.ntp时间是否同步
>
> 2.是否之前部署过后进行了helm delete，如果是，需要手动删除openstack namespace下，mariaDB相关的configmap和pvc，删除后重新部署即可





