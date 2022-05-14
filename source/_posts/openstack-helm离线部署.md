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
> Linux发行版为centos7，openstack版本为train版（train以上版本无法完全适配centos7 3.10 kernel）

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

3.为节点添加label（3控制同时作为控制节点和计算节点，单独的node4作为单独计算节点，所有节点默认都是存储节点）

```
kubectl label nodes knode1 ceph-control-plane=enabled openstack-control-plane=enabled  openvswitch=enabled openstack-compute-node=enabled
kubectl label nodes knode2 ceph-control-plane=enabled openstack-control-plane=enabled  openvswitch=enabled openstack-compute-node=enabled
kubectl label nodes knode3 ceph-control-plane=enabled openstack-control-plane=enabled  openvswitch=enabled openstack-compute-node=enabled
kubectl label nodes knode4 openstack-compute-node=enabled openvswitch=enabled 
```



# 二.安装rook-ceph

1.安装rook-ceph-operator

```
helm install --create-namespace --namespace rook-ceph rook-ceph ./charts/rook-ceph-0.0.1.tgz  --set image.tag=v1.8.8
# wait for operator start
sleep 10
```

2.创建ceph cluster、storageClass

```
helm install --create-namespace --namespace rook-ceph rook-ceph-cluster ./charts/rook-ceph-cluster-0.0.1.tgz --set cephClusterSpec.dashboard.enabled=false
```

> 删除时，需要先执行```kubectl -n rook-ceph patch cephcluster rook-ceph --type merge -p '{"spec":{"cleanupPolicy":{"confirmation":"yes-really-destroy-data"}}}'```
>

3.等待集群创建完成

```
kubectl  wait --for=condition=Ready -n rook-ceph cephcluster rook-ceph --timeout=1800s
 
# 等待osd pod正常创建并启动
sleep 180

kubectl get pod -n rook-ceph # 确认所有pod都处于Running或Complete状态，确保4个osd成功创建（集群Ready后，OSD不一定完全启动），创建成功后才能进行下一步
# 查看ceph集群状态
kubectl exec -it -n rook-ceph `kubectl  get pod -n rook-ceph -owide|grep tools|awk '{ print $1 }'` -- ceph -s
```

> 1.ceph osd无法启动？
>
> 查看prepare osd job pod的日志，确认各节点有一块以上未使用的磁盘，operator轮询过程可能比较慢，可能需要等待5分钟以上才能创建出osd
>
> 2.新增节点osd未被添加？
>
> rook operator会以周期频率检查新加入节点，如果长时间未被添加，手动重启rook operator pod，1分钟左右新节点会被加入

# 三.安装Registry、ChartMuseum

1.安装并配置docker-registry

```
helm install docker-registry --create-namespace --namespace registry ./charts/docker-registry-2.1.0.tgz --set persistence.size=60Gi
kubectl get pod  -n registry |grep 1/1 # 确认registry pod成功创建，成功后才能进行下一步
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
sleep 30
kubectl  wait --for=condition=Available -n registry deploy docker-registry --timeout=120s
```

> 测试：
>
> docker tag rook/ceph:v1.8.8 docker-registry.registry.svc.cluster.local/ceph:v1.8.8
>
> docker push docker-registry.registry.svc.cluster.local/ceph:v1.8.8

3.导入镜像到registry

```
# 添加kubedns，所有节点都需要配置该nameserver，否则kubelet无法拉取镜像，node节点如果无法执行kubectl命令，则需要从master节点拷贝
NAMESERVER=`kubectl get svc -n kube-system|awk '{ print $3 }'|grep -v IP`
echo "nameserver $NAMESERVER" > /etc/resolv.conf


# 修改Kubelet ImageGC配置，防止kubelet删除节点镜像
echo "imageGCHighThresholdPercent: 100" >> /var/lib/kubelet/config.yaml
systemctl restart kubelet

docker load -q < images/components/all_components_images.tar.gz
docker images | grep docker-registry.registry.svc.cluster.local | awk '{print "docker push -q "$1":"$2}'|sh

# 复原kubelet配置
sed -i '/imageGCHighThresholdPercent/d'  /var/lib/kubelet/config.yaml
systemctl restart kubelet
sleep 30
```

4.安装并配置chartmuseum

```
helm install chartmuseum --create-namespace --namespace registry ./charts/chartmuseum-3.7.0.tgz
kubectl  wait --for=condition=Available -n registry deploy chartmuseum --timeout=120s
helm repo add chartmuseum http://chartmuseum.registry.svc.cluster.local:8080/
helm repo update
```

5.安装并配置openstackclient

```
yum install -y python3
mkdir -p /tmp/pip_packs/
tar -zxf ./pip_packs/openstackclient_pip_packs.tar.gz -C /tmp/pip_packs/
sudo -H -E pip3 install  --no-index --find-links=/tmp/pip_packs/ cmd2 python-openstackclient python-heatclient --ignore-installed

sudo -H mkdir -p /etc/openstack
sudo -H chown -R $(id -un): /etc/openstack
FEATURE_GATE="tls"; if [[ ${FEATURE_GATES//,/ } =~ (^|[[:space:]])${FEATURE_GATE}($|[[:space:]]) ]]; then
  tee /etc/openstack/clouds.yaml << EOF
  clouds:
    openstack_helm:
      region_name: RegionOne
      identity_api_version: 3
      volume_api_version: 3
      cacert: /etc/openstack-helm/certs/ca/ca.pem
      auth:
        username: 'admin'
        password: 'password'
        project_name: 'admin'
        project_domain_name: 'default'
        user_domain_name: 'default'
        auth_url: 'https://keystone-api.openstack.svc.cluster.local:5000/v3'
EOF
else
  tee /etc/openstack/clouds.yaml << EOF
  clouds:
    openstack_helm:
      region_name: RegionOne
      identity_api_version: 3
      volume_api_version: 3
      auth:
        username: 'admin'
        password: 'password'
        project_name: 'admin'
        project_domain_name: 'default'
        user_domain_name: 'default'
        auth_url: 'http://keystone-api.openstack.svc.cluster.local:5000/v3'
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
helm install mariadb --create-namespace --namespace openstack chartmuseum/mariadb --set volume.enabled=true --set volume.class_name=ceph-block --set volume.size=10Gi --set pod.replicas.server=3

# 等待3个pod启动
kubectl  wait --for=condition=Ready -n openstack pods/mariadb-server-2 --timeout 500s
kubectl  wait --for=condition=Ready -n openstack pods/mariadb-server-1 --timeout 500s
kubectl  wait --for=condition=Ready -n openstack pods/mariadb-server-0 --timeout 500s
```

> Pod一直无法启动？
>
> 一般情况下是mariaDB集群时间采样与实际时间不符导致，需要检查以下两点：
>
> 1.ntp时间是否同步
>
> 2.是否之前部署过后进行了helm delete，如果是，需要手动删除openstack namespace下，mariaDB相关的configmap和pvc，删除后重新部署即可

3.部署RabbitMQ

```
helm upgrade --install rabbitmq chartmuseum/rabbitmq \
    --namespace=openstack \
    --set volume.enabled=true \
    --set pod.replicas.server=2 \
    --set volume.class_name=ceph-block
```

4.部署Memcached

```
helm upgrade --install memcached chartmuseum/memcached \
    --namespace=openstack

# 等待pod启动
kubectl  wait --for=condition=Available -n openstack deployment/memcached-memcached --timeout=300s
```



# 四.部署openstack相关组件

1.部署keystone

```
helm upgrade --install keystone chartmuseum/keystone     --namespace=openstack     --set pod.replicas.api=2 --timeout 1800s

kubectl  wait --for=condition=Available -n openstack deploy/keystone-api --timeout=300s
# 验证
export OS_CLOUD=openstack_helm
openstack endpoint list
```

2.部署glance

```
helm upgrade --install glance chartmuseum/glance \
  --namespace=openstack --timeout 1800s
  
# 查看pod状态
kubectl  get pod -n openstack|grep glance|grep -v Completed
# 验证
export OS_CLOUD=openstack_helm
openstack image create "Cirros 0.3.5 64-bit" --min-disk 1 --disk-format qcow2 --file ./images/cirros-0.3.5-x86_64-disk.img --container-format bare  --public
openstack image list
```

3.部署cinder

```
CEPHKEY=`kubectl get secret -n rook-ceph rook-ceph-admin-keyring -ojsonpath='{.data.keyring}'|base64 -d|grep key|awk '{ print $3 }'`
MON_LIST=($(kubectl  get svc -n rook-ceph|grep mon|awk '{ print $1 }'))
cat << EOF | sudo tee /tmp/cinder_ceph.yaml
# Note: This yaml file serves as an example for overriding the manifest
# to enable additional externally managed Ceph Cinder backend.
# libvirt/values_overrides/cinder-external-ceph-backend.yaml in repo
# openstack-helm-infra is also needed for the attachment of ceph volumes.
---
ceph_client:
  enable_external_ceph_backend: True
  external_ceph:
    rbd_user: admin
    rbd_user_keyring: $CEPHKEY
    conf:
      global:
        mon_host: "${MON_LIST[0]}.rook-ceph.svc.cluster.local:6789,${MON_LIST[1]}.rook-ceph.svc.cluster.local:6789,${MON_LIST[2]}.rook-ceph.svc.cluster.local:6789"
      client.admin:
        keyring: /etc/ceph/ceph.client.admin.keyring
conf:
  ceph:
    admin_keyring: $CEPHKEY
  cinder:
    DEFAULT:
      enabled_backends: "rook-ceph"
      backup_ceph_user: "admin"
  backends:
    rook-ceph:
      volume_driver: cinder.volume.drivers.rbd.RBDDriver
      volume_backend_name: rook-ceph
      rbd_pool: cinder.rook
      rbd_ceph_conf: "/etc/ceph/external-ceph.conf"
      rbd_flatten_volume_from_snapshot: False
      report_discard_supported: True
      rbd_max_clone_depth: 5
      rbd_store_chunk_size: 4
      rados_connect_timeout: -1
      rbd_user: admin
      rbd_secret_uuid: 3f0133e4-8384-4743-9473-fecacc095c74
      image_volume_cache_enabled: True
      image_volume_cache_max_size_gb: 200
      image_volume_cache_max_count: 50
  backup:
    external_ceph_rbd:
      enabled: true
      admin_keyring: $CEPHKEY
      conf:
        global:
          mon_host: "${MON_LIST[0]}.rook-ceph.svc.cluster.local:6789,${MON_LIST[1]}.rook-ceph.svc.cluster.local:6789,${MON_LIST[2]}.rook-ceph.svc.cluster.local:6789"
        client.admin:
          keyring: /etc/ceph/ceph.client.admin.keyring
...

EOF

cat <<EOF | sudo tee /tmp/ceph-ceph.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ceph-etc
  namespace: openstack
data:
  ceph.conf: |
    [global]
    mon_host = ${MON_LIST[0]}.rook-ceph.svc.cluster.local:6789,${MON_LIST[1]}.rook-ceph.svc.cluster.local:6789,${MON_LIST[2]}.rook-ceph.svc.cluster.local:6789

    [client.admin]
    keyring = /etc/ceph/ceph.client.admin.keyring
EOF
kubectl apply -f /tmp/ceph-ceph.yaml


helm upgrade --install cinder chartmuseum/cinder \
  --namespace=openstack \
  --values=/tmp/cinder_ceph.yaml --timeout 1800s
kubectl  wait --for=condition=Available -n openstack deploy/cinder-api --timeout=300s
# 验证
export OS_CLOUD=openstack_helm
openstack volume type list
openstack volume type list --default
```

4.部署openvswitch

```
helm upgrade --install openvswitch chartmuseum/openvswitch \
  --namespace=openstack
# 验证, 查看daemonSet正常启动
[root@master1 ~]# kubectl  get ds -n openstack
NAME                   DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR         AGE
openvswitch-db         4         4         4       4            4           openvswitch=enabled   2m19s
openvswitch-vswitchd   4         4         4       4            4           openvswitch=enabled   2m19s
```

5.部署Libvirt

```
CEPHKEY=`kubectl get secret -n rook-ceph rook-ceph-admin-keyring -ojsonpath='{.data.keyring}'|base64 -d|grep key|awk '{ print $3 }'`
cat << EOF | sudo tee /tmp/libvirt_ceph.yaml
# Note: This yaml file serves as an example for overriding the manifest
# to enable additional externally managed Ceph Cinder backend. When additional
# externally managed Ceph Cinder backend is provisioned as shown in
# cinder/values_overrides/external-ceph-backend.yaml of repo openstack-helm,
# below override is needed to store the secret key of the cinder user in
# libvirt.
---
conf:
  ceph:
    enabled: true
    admin_keyring: $CEPHKEY
    cinder:
      user: "admin"
      keyring: $CEPHKEY
      secret_uuid: 3f0133e4-8384-4743-9473-fecacc095c74
      external_ceph:
        enabled: true
        user: admin
        secret_uuid: 3f0133e4-8384-4743-9473-fecacc095c74
        user_secret_name: cinder-volume-external-rbd-keyring
...

EOF
helm upgrade --install libvirt chartmuseum/libvirt \
  --namespace=openstack --values=/tmp/libvirt_ceph.yaml --timeout 1800s
```

> 在没有安装neutron的情况下，libvirt是无法完全启动的，pod处于init状态，查看init日志，kubectl logs -n openstack libvirt-libvirt-default-7trpd -c init，可以看到Entrypoint WARNING: 2022/05/10 02:55:23 entrypoint.go:72: Resolving dependency Pod on same host with labels map[application:neutron component:neutron-ovs-agent] in namespace openstack failed: Found no pods matching labels: map[application:neutron component:neutron-ovs-agent]
>
> 稍后在neutron部署完成后，在验证libvirt是否正常启动

6.部署nova、neutron（Compute Kit）

**需要注意neutron外部网卡的配置！！**

```

# 计算组件（Compute Kit）安装过程会比较久（大约20~40分钟），可以通过循环检测判定安装状态
CEPHKEY=`kubectl get secret -n rook-ceph rook-ceph-admin-keyring -ojsonpath='{.data.keyring}'|base64 -d|grep key|awk '{ print $3 }'`
cat << EOF | sudo tee /tmp/nova_ceph.yaml
---
pod:
  replicas:
    osapi: 2
    conductor: 2
conf:
  ceph:
    enabled: true
    admin_keyring: $CEPHKEY
    cinder:
      user: "admin"
      keyring: $CEPHKEY
      secret_uuid: 3f0133e4-8384-4743-9473-fecacc095c74
  nova:
    libvirt:
      rbd_secret_uuid: 3f0133e4-8384-4743-9473-fecacc095c74
      rbd_user: admin
...
EOF
#NOTE: Deploy nova
if [ "x$(systemd-detect-virt)" == "xnone" ]; then
  echo 'OSH is not being deployed in virtualized environment'
  helm upgrade --install nova chartmuseum/nova \
      --namespace=openstack \
      --values=/tmp/nova_ceph.yaml \
      --set bootstrap.wait_for_computes.enabled=true \
      --timeout 1800s
else
  echo 'OSH is being deployed in virtualized environment, using qemu for nova'
  helm upgrade --install nova chartmuseum/nova \
      --namespace=openstack \
      --values=/tmp/nova_ceph.yaml \
      --set bootstrap.wait_for_computes.enabled=true \
      --set conf.nova.libvirt.virt_type=qemu \
      --set conf.nova.libvirt.cpu_mode=none \
      --timeout 1800s
fi

#NOTE: Deploy placement
helm upgrade --install placement chartmuseum/placement \
    --namespace=openstack --timeout 1800s

#NOTE: Deploy neutron

# ==================注意=======================
PUBLIC_NET_NIC=eth0 # 设置public网络的桥接网卡
GATEWAY=10.10.1.1 # 设置外部网关
# ==================注意=======================

tee /tmp/neutron.yaml << EOF
network:
  interface:
    tunnel: docker0
pod:
  replicas:
    server: 2
conf:
  auto_bridge_add:
    br-ex: $PUBLIC_NET_NIC
  neutron:
    DEFAULT:
      l3_ha: False
      max_l3_agents_per_router: 1
      l3_ha_network_type: vxlan
      dhcp_agents_per_network: 1
  plugins:
    ml2_conf:
      ml2_type_flat:
        flat_networks: public
    openvswitch_agent:
      agent:
        tunnel_types: vxlan
      ovs:
        bridge_mappings: public:br-ex
    linuxbridge_agent:
      linux_bridge:
        bridge_mappings: public:br-ex
EOF
helm upgrade --install neutron chartmuseum/neutron \
    --namespace=openstack \
    --values=/tmp/neutron.yaml --timeout 1800s

# 若public网卡为唯一网卡，neutron安装完成后会丢失默认路由，需要重新添加
ip route |grep default ||  ip route add default via $GATEWAY dev br-ex

# 等待所有pod启动完成，时间大约在20分钟以内
kubectl get pod -n openstack |grep -v Running |grep -v Completed
# 验证
export OS_CLOUD=openstack_helm
[root@knode1 ~]# openstack service list

+----------------------------------+-----------+-----------+
| ID                               | Name      | Type      |
+----------------------------------+-----------+-----------+
| 4b8e84c1b0964d858b9c64291c06e79e | cinder    | volumev3  |
| 524d333166fe41359178ffbd4e6c216e | keystone  | identity  |
| 53549bf12e134837a9fb8606a5c1a027 | placement | placement |
| 68a103d3bfde490da4be9c66b09717cb | neutron   | network   |
| 9362ca20a79b469aa76b874dc2e06c3b | glance    | image     |
| c695d0d4ca5b463fbfb6b8e7379f459e | nova      | compute   |
+----------------------------------+-----------+-----------+
[root@knode1 ~]# openstack compute service list
+----+----------------+---------------------------------+----------+---------+-------+----------------------------+
| ID | Binary         | Host                            | Zone     | Status  | State | Updated At                 |
+----+----------------+---------------------------------+----------+---------+-------+----------------------------+
|  4 | nova-scheduler | nova-scheduler-69cbb6b779-59b28 | internal | enabled | up    | 2022-05-13T14:55:04.000000 |
|  9 | nova-conductor | nova-conductor-6477bc8d6c-zbrw2 | internal | enabled | up    | 2022-05-13T14:54:57.000000 |
| 12 | nova-conductor | nova-conductor-6477bc8d6c-rvwrq | internal | enabled | up    | 2022-05-13T14:55:02.000000 |
| 18 | nova-compute   | knode1                          | nova     | enabled | up    | 2022-05-13T14:54:57.000000 |
| 20 | nova-compute   | knode3                          | nova     | enabled | up    | 2022-05-13T14:55:02.000000 |
| 21 | nova-compute   | knode2                          | nova     | enabled | up    | 2022-05-13T14:55:00.000000 |
| 23 | nova-compute   | knode4                          | nova     | enabled | up    | 2022-05-13T14:55:00.000000 |
+----+----------------+---------------------------------+----------+---------+-------+----------------------------+
[root@knode1 ~]# openstack network agent list
+--------------------------------------+--------------------+--------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host   | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+--------+-------------------+-------+-------+---------------------------+
| 0c2397e8-0afd-47a7-a554-b67b089e1adf | Metadata agent     | knode3 | None              | :-)   | UP    | neutron-metadata-agent    |
| 327e1437-0963-4990-8009-3ecb259d3c80 | DHCP agent         | knode2 | nova              | :-)   | UP    | neutron-dhcp-agent        |
| 3b554200-02ec-41d5-81da-ea05741337e6 | Metadata agent     | knode1 | None              | :-)   | UP    | neutron-metadata-agent    |
| 55ed67d8-77dd-4192-8022-48f9c7e26969 | DHCP agent         | knode1 | nova              | :-)   | UP    | neutron-dhcp-agent        |
| 624755f4-8031-4b5d-830f-6c2ec656b111 | Metadata agent     | knode2 | None              | :-)   | UP    | neutron-metadata-agent    |
| 8e1f1825-7ab5-4589-8c43-81e2395c6fe8 | Open vSwitch agent | knode1 | None              | :-)   | UP    | neutron-openvswitch-agent |
| 8f6af6dc-4eac-4a9f-8c46-b8ad1c3bd64e | Open vSwitch agent | knode3 | None              | :-)   | UP    | neutron-openvswitch-agent |
| 9a1d6634-cb30-4011-92cb-9e07ae981d05 | L3 agent           | knode2 | nova              | :-)   | UP    | neutron-l3-agent          |
| bf372b1f-4b01-4fa4-8309-9d6975bb8373 | L3 agent           | knode1 | nova              | :-)   | UP    | neutron-l3-agent          |
| c8073699-e64f-4dea-b1f4-a82d2f97c46e | DHCP agent         | knode3 | nova              | :-)   | UP    | neutron-dhcp-agent        |
| ce01381b-0843-4af8-bf10-d443dded980e | Open vSwitch agent | knode4 | None              | :-)   | UP    | neutron-openvswitch-agent |
| d9f47dc5-22d5-4556-b1eb-7e9c8a0c7282 | L3 agent           | knode3 | nova              | :-)   | UP    | neutron-l3-agent          |
| ee1d334d-88c3-45c1-83aa-c6bd08449d2d | Open vSwitch agent | knode2 | None              | :-)   | UP    | neutron-openvswitch-agent |
+--------------------------------------+--------------------+--------+-------------------+-------+-------+---------------------------+
[root@knode1 ~]# openstack hypervisor list
+----+---------------------+-----------------+-------------+-------+
| ID | Hypervisor Hostname | Hypervisor Type | Host IP     | State |
+----+---------------------+-----------------+-------------+-------+
|  2 | knode1              | QEMU            | 10.10.1.100 | up    |
|  5 | knode3              | QEMU            | 10.10.1.120 | up    |
|  8 | knode2              | QEMU            | 10.10.1.110 | up    |
|  9 | knode4              | QEMU            | 10.10.1.130 | up    |
+----+---------------------+-----------------+-------------+-------+
```

7.安装heat

```
helm upgrade --install heat chartmuseum/heat \
  --namespace=openstack

# 验证
export OS_CLOUD=openstack_helm
openstack service list
openstack endpoint list

openstack --os-interface public orchestration service list
+------------------------------+-------------+--------------------------------------+-------------+--------+----------------------------+--------+
| Hostname                     | Binary      | Engine ID                            | Host        | Topic  | Updated At                 | Status |
+------------------------------+-------------+--------------------------------------+-------------+--------+----------------------------+--------+
| heat-engine-5bc7886f47-vps7v | heat-engine | df962df7-3829-4bff-8c77-3df5b7cadd49 | heat-engine | engine | 2022-05-13T15:11:50.000000 | up     |
| heat-engine-5bc7886f47-khr52 | heat-engine | 5358bba9-aa70-4bb2-be57-25138763dd16 | heat-engine | engine | 2022-05-13T15:11:51.000000 | up     |
+------------------------------+-------------+--------------------------------------+-------------+--------+----------------------------+--------+
```

8.安装horizon（可选）

```
helm upgrade --install horizon chartmuseum/horizon \
  --namespace=openstack
```



# 五.注意事项

**1.步骤中wait方式等待可能会不准确，即pod未就绪，但deploy状态已就绪，需要确认pod处于Running后，再进行对应验证操作，否则会失败**

**2.在placement安装过程中，nova相关组件可能会出现认证错误，在neutron完成安装并启动完成前可以忽视这些错误**
