---
title: kubeadm部署生产k8s
date: 2022-04-15 15:59:59
tags: k8s
---





# 一.环境准备

| IP              | 主机名/类型 | ApiServer端口号 |
| --------------- | ----------- | --------------- |
| 192.168.122.213 | knode1      | 6443            |
| 192.168.122.74  | knode2      | 6443            |
| 192.168.122.18  | knode3      | 6443            |
| 192.168.122.150 | VIP         | 7443            |

1.基础参数配置（所有节点）

设置主机名：

`hostnamectl set-hostname node1`

生成密钥、免密登陆：

`ssh-keygen`，`ssh-copy-id` 

关闭防火墙：

```
systemctl stop firewalld
systemctl disable firewalld
```

加载 `br_netfilter` 模块  ，设置桥接。

```bash
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

2.安装并配置docker（所有节点）

```
sudo yum remove docker                   docker-client                   docker-client-latest                   docker-common                   docker-latest                   docker-latest-logrotate                   docker-logrotate                   docker-engine
sudo yum install -y yum-utils
sudo yum-config-manager     --add-repo     https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```



# 二.通过kubeadm部署融合生产环境（3节点）

1.安装kubeadm（所有节点）

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

2.初始化环境变量（所有节点）

```
APISERVER_VIP=192.168.122.150

APISERVER_DEST_PORT=7443

APISERVER_SRC_PORT=6443

HOST1_ID=knode1

HOST1_ADDRESS=192.168.122.213

HOST2_ID=knode2

HOST2_ADDRESS=192.168.122.74

HOST3_ID=knode3

HOST3_ADDRESS=192.168.122.18
```

3.为kube-api配置负载均衡（keepalive+haproxy）（所有节点）

keepalive& haproxy配置：

```
########### keepalived ###########
sudo yum install -y keepalived
sudo cat <<EOF | sudo tee /etc/keepalived/keepalived.conf
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    authentication {
        auth_type PASS
        auth_pass 42
    }
    virtual_ipaddress {
        ${APISERVER_VIP} # VIP地址
    }
    track_script {
        check_apiserver
    }
}
EOF

sudo cat <<EOF | sudo tee /etc/keepalived/check_apiserver.sh
#!/bin/sh

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

curl --silent --max-time 2 --insecure https://localhost:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://localhost:${APISERVER_DEST_PORT}/"
if ip addr | grep -q ${APISERVER_VIP}; then
    curl --silent --max-time 2 --insecure https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/"
fi
EOF

########### haproxy ###########
sudo yum install -y haproxy
sudo cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg
# /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 1
    timeout http-request    10s
    timeout queue           20s
    timeout connect         5s
    timeout client          20s
    timeout server          20s
    timeout http-keep-alive 10s
    timeout check           10s

#---------------------------------------------------------------------
# apiserver frontend which proxys to the control plane nodes
#---------------------------------------------------------------------
frontend apiserver
    bind *:${APISERVER_DEST_PORT}
    mode tcp
    option tcplog
    default_backend apiserver

#---------------------------------------------------------------------
# round robin balancing for apiserver
#---------------------------------------------------------------------
backend apiserver
    option httpchk GET /healthz
    http-check expect status 200
    mode tcp
    option ssl-hello-chk
    balance     roundrobin
        server ${HOST1_ID} ${HOST1_ADDRESS}:${APISERVER_SRC_PORT} check
        server ${HOST2_ID} ${HOST2_ADDRESS}:${APISERVER_SRC_PORT} check
        server ${HOST3_ID} ${HOST3_ADDRESS}:${APISERVER_SRC_PORT} check
EOF
systemctl enable haproxy --now
systemctl enable keepalived --now
```

4.初始化控制平面（第一个控制节点）

```
########### init first control ###########

sudo kubeadm init --control-plane-endpoint "${APISERVER_VIP}:${APISERVER_DEST_PORT}" --upload-certs  --pod-network-cidr=10.244.0.0/16 --kubernetes-version 1.23.5
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

5.安装flannel网络插件（vxlan）（第一个控制节点）

```
sudo cat <<EOF | sudo tee /root/flannel.yaml
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.flannel.unprivileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
    seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
    apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
    apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
spec:
  privileged: false
  volumes:
  - configMap
  - secret
  - emptyDir
  - hostPath
  allowedHostPaths:
  - pathPrefix: "/etc/cni/net.d"
  - pathPrefix: "/etc/kube-flannel"
  - pathPrefix: "/run/flannel"
  readOnlyRootFilesystem: false
  # Users and groups
  runAsUser:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  # Privilege Escalation
  allowPrivilegeEscalation: false
  defaultAllowPrivilegeEscalation: false
  # Capabilities
  allowedCapabilities: ['NET_ADMIN', 'NET_RAW']
  defaultAddCapabilities: []
  requiredDropCapabilities: []
  # Host namespaces
  hostPID: false
  hostIPC: false
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  # SELinux
  seLinux:
    # SELinux is unused in CaaSP
    rule: 'RunAsAny'
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups: ['extensions']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames: ['psp.flannel.unprivileged']
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni-plugin
       #image: flannelcni/flannel-cni-plugin:v1.0.1 for ppc64le and mips64le (dockerhub limitations may apply)
        image: rancher/mirrored-flannelcni-flannel-cni-plugin:v1.0.1
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
       #image: flannelcni/flannel:v0.17.0 for ppc64le and mips64le (dockerhub limitations may apply)
        image: rancher/mirrored-flannelcni-flannel:v0.17.0
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
       #image: flannelcni/flannel:v0.17.0 for ppc64le and mips64le (dockerhub limitations may apply)
        image: rancher/mirrored-flannelcni-flannel:v0.17.0
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
        - name: xtables-lock
          mountPath: /run/xtables.lock
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /opt/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
EOF
kubectl  apply -f flannel.yaml
```

6.添加额外的控制节点

查看第一个控制节点状态：

```
[root@knode1 ~]# kubectl get pod -n kube-system -w
NAME                             READY   STATUS    RESTARTS   AGE
coredns-64897985d-4hjxc          1/1     Running   0          2m6s
coredns-64897985d-9x4lz          1/1     Running   0          2m6s
etcd-knode1                      1/1     Running   1          2m15s
kube-apiserver-knode1            1/1     Running   1          2m15s
kube-controller-manager-knode1   1/1     Running   0          2m14s
kube-flannel-ds-v45xx            1/1     Running   0          45s
kube-proxy-b55jc                 1/1     Running   0          2m6s
kube-scheduler-knode1            1/1     Running   1          2m15s
```

获取密钥信息（在第一个控制节点执行）：

```
FIRSTCONTROLNODE=192.168.122.213
TOKEN=`kubeadm token create`
DISCOVERYTOKEHASH=`openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'`
CERTKEY=`kubeadm init phase upload-certs --upload-certs --one-output|grep -v upload-certs`
```

加入额外控制节点（在需要加入的额外控制节点上执行）：

```
sudo kubeadm join ${FIRSTCONTROLNODE}:6443 --token ${TOKEN} --discovery-token-ca-cert-hash sha256:${DISCOVERYTOKEHASH} --control-plane --certificate-key ${CERTKEY}
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

加入额外node节点（在需要加入的额外node节点上执行）：

```
sudo kubeadm join ${FIRSTCONTROLNODE}:6443 --token ${TOKEN} --discovery-token-ca-cert-hash sha256:${DISCOVERYTOKEHASH}
```





# 三.验证

在任意控制节点执行：

```
[root@knode1 ~]# kubectl get pod -n kube-system  -owide
NAME                             READY   STATUS    RESTARTS        AGE     IP                NODE     NOMINATED NODE   READINESS GATES
coredns-64897985d-9fhq7          1/1     Running   0               13m     10.244.0.2        knode1   <none>           <none>
coredns-64897985d-v2pg4          1/1     Running   0               13m     10.244.0.3        knode1   <none>           <none>
etcd-knode1                      1/1     Running   4               13m     192.168.122.213   knode1   <none>           <none>
etcd-knode2                      1/1     Running   0               6m49s   192.168.122.74    knode2   <none>           <none>
etcd-knode3                      1/1     Running   0               64s     192.168.122.18    knode3   <none>           <none>
kube-apiserver-knode1            1/1     Running   4               13m     192.168.122.213   knode1   <none>           <none>
kube-apiserver-knode2            1/1     Running   0               6m53s   192.168.122.74    knode2   <none>           <none>
kube-apiserver-knode3            1/1     Running   0               67s     192.168.122.18    knode3   <none>           <none>
kube-controller-manager-knode1   1/1     Running   4 (6m39s ago)   13m     192.168.122.213   knode1   <none>           <none>
kube-controller-manager-knode2   1/1     Running   0               6m52s   192.168.122.74    knode2   <none>           <none>
kube-controller-manager-knode3   1/1     Running   0               67s     192.168.122.18    knode3   <none>           <none>
kube-flannel-ds-9x9ls            1/1     Running   0               68s     192.168.122.18    knode3   <none>           <none>
kube-flannel-ds-vdzgk            1/1     Running   0               13m     192.168.122.213   knode1   <none>           <none>
kube-flannel-ds-xdwcj            1/1     Running   1 (6m16s ago)   6m53s   192.168.122.74    knode2   <none>           <none>
kube-proxy-4zqdz                 1/1     Running   0               68s     192.168.122.18    knode3   <none>           <none>
kube-proxy-grwrp                 1/1     Running   0               13m     192.168.122.213   knode1   <none>           <none>
kube-proxy-pxzvj                 1/1     Running   0               6m53s   192.168.122.74    knode2   <none>           <none>
kube-scheduler-knode1            1/1     Running   5 (6m39s ago)   13m     192.168.122.213   knode1   <none>           <none>
kube-scheduler-knode2            1/1     Running   0               6m52s   192.168.122.74    knode2   <none>           <none>
kube-scheduler-knode3            1/1     Running   0               67s     192.168.122.18    knode3   <none>           <none>
```





# 四.离线部署

1.如果联机部署需要配置代理访问，包括镜像源和yum源代理，如果本地离线部署，需要提前准备好本地镜像仓库与本地yum源：

kubernetes yum源参考：

```
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
proxy=http://knode1:8090
```

下载离线镜像：

```
kubeadm config images list
kubeadm config images pull
docker pull rancher/mirrored-flannelcni-flannel:v0.17.0
docker pull rancher/mirrored-flannelcni-flannel-cni-plugin:v1.0.1
```

2.打包镜像

```
IMAGES_LIST=($(docker  images   | sed  '1d' | awk  '{print $1":"$2}'))
docker save ${IMAGES_LIST[*]}  -o  all-images.tar.gz
```

3.在其他节点导入：

```
docker load < all-images.tar.gz
```
