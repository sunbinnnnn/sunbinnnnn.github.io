---
title: CSI plugin
date: 2021-04-20 16:37:27
tags: k8s
---

在了解k8s的CSI plugin编写前，我们需要先了解下有关K8S的持久化存储机制。

<!--more-->



### 理解k8s持久化存储

在k8s中，持久化存储采用PV和PVC进行绑定的的方式进行管理。

*PV（PersistentVolume）：*存储卷对象映射，一般由管理员手动创建或通过存储插件（External Provisioner）创建。示例：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem # K8S支持两种volumeMode：Filesystem和Block
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle # 三种策略可选：Retain\Recycle\Delete，只有NFS和HostPath支持Recycle（纯调用rm -rf命令进行文件系统级别删除）
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```



*PVC（PersistentVolumeClaim）*：存储卷声明，一般由开发人员定义，对于支持Dynamic Provisioning的存储类型，通过对PVC的声明（可以在pod中完成），可以让PersistentVolumeController找到一块合适的PV与PVC进行bound操作。示例：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 30Gi
```



这种绑定操作可以是**静态的（Static Provisioning）**，也可以是**动态的（Dynamic Provisioning）**

首先说静态，通过静态方式进行时，由管理员创建PV，通过PersistentVolumeController，k8s可以完成PV和PVC的绑定，PersistentVolumeController(`pkg/controller/volume/persistentvolume/pv_controller.go`)存在一个控制循环，不断遍历所有可用状态的PV，尝试与PVC进行绑定（Bound）操作，绑定成功后，则为声明该PVC的Pod提供存储服务。

#### PV和PVC绑定调度流程：

当PVC被声明出来时（单独声明 or statefulSet），会被cache.Controller watch到，并开始执行`syncClaim`函数：

```go
func (ctrl *PersistentVolumeController) syncClaim(claim *v1.PersistentVolumeClaim) error {
	klog.V(4).Infof("synchronizing PersistentVolumeClaim[%s]: %s", claimToClaimKey(claim), getClaimStatusForLogging(claim))

	// Set correct "migrated-to" annotations on PVC and update in API server if
	// necessary
	newClaim, err := ctrl.updateClaimMigrationAnnotations(claim)
	if err != nil {
		// Nothing was saved; we will fall back into the same
		// condition in the next call to this method
		return err
	}
	claim = newClaim

	if !metav1.HasAnnotation(claim.ObjectMeta, pvutil.AnnBindCompleted) {
		return ctrl.syncUnboundClaim(claim)
	} else {
		return ctrl.syncBoundClaim(claim)
	}
}
```

通过`pv.kubernetes.io/bind-completed` annotation来判断pvc是否已经完成bound操作，如果该PVC未进行bound操作，则调用`syncUnboundClaim`进行bound操作。

在进行`syncUnboundClaim`前，首先会确认PVC是否定义了**延迟绑定**策略：

```go
// IsDelayBindingMode checks if claim is in delay binding mode.
func IsDelayBindingMode(claim *v1.PersistentVolumeClaim, classLister storagelisters.StorageClassLister) (bool, error) {
	className := storagehelpers.GetPersistentVolumeClaimClass(claim)
	if className == "" {
		return false, nil
	}

	class, err := classLister.Get(className)
	if err != nil {
		if apierrors.IsNotFound(err) {
			return false, nil
		}
		return false, err
	}

	if class.VolumeBindingMode == nil {
		return false, fmt.Errorf("VolumeBindingMode not set for StorageClass %q", className)
	}

	return *class.VolumeBindingMode == storage.VolumeBindingWaitForFirstConsumer, nil
}
```

延迟绑定主要用在Local PersistentVolume的情况下，当采用本地卷作为持久化卷时，如果PVC和PV即时绑定，则可能在pod启动的节点上找不到PV，mount过程会失败，而延迟绑定则将PVC和PV的绑定延后到Pod 调度器中，从而使Volume卷可以被正常挂载到Pod上。

之后执行PV查找过程，首先从`pvIndex`中按照`AccessModes`找到所有符合的PV：

```go
allPossibleModes := pvIndex.allPossibleMatchingAccessModes(claim.Spec.AccessModes)
```

例如PVC请求的PV的AccessMode是`ReadWriteOnce`，则包含`ReadWriteOnce`的PV都会被检索出。

之后通过调用`FindMatchingVolume`方法找到最合适的PV。

这里的逻辑是通过遍历符合AccessMode的所有PV，首先判定PV是否已经被其他PVC预绑定（pre-bound）或已经被绑定：

```go
// ...
if volume.Spec.ClaimRef != nil && !IsVolumeBoundToClaim(volume, claim) {
    continue
}
// ...
func IsVolumeBoundToClaim(volume *v1.PersistentVolume, claim *v1.PersistentVolumeClaim) bool {
	if volume.Spec.ClaimRef == nil {
		return false
	}
	if claim.Name != volume.Spec.ClaimRef.Name || claim.Namespace != volume.Spec.ClaimRef.Namespace {
		return false
	}
	if volume.Spec.ClaimRef.UID != "" && claim.UID != volume.Spec.ClaimRef.UID {
		return false
	}
	return true
}
```

当开启了 延迟绑定后，PV将会被直接跳过，交给Pod调度器进行调度：

```go
if node == nil && delayBinding {
    // PV controller does not bind this claim.
    // Scheduler will handle binding unbound volumes
    // Scheduler path will have node != nil
    continue
}
```

最后会检查PV的状态是否处于 Available 、PVC中定义的labelSelector是否符合要求以及StorageClass是否符合（默认都为空，则为符合），不符合则跳过：

```go
if volume.Status.Phase != v1.VolumeAvailable {
    // We ignore volumes in non-available phase, because volumes that
    // satisfies matching criteria will be updated to available, binding
    // them now has high chance of encountering unnecessary failures
    // due to API conflicts.
    continue
} else if selector != nil && !selector.Matches(labels.Set(volume.Labels)) {
	continue
}
if storagehelpers.GetPersistentVolumeClass(volume) != requestedClass {
	continue
}
```

以上都完毕后，从所有的符合条件的PV中找到符合PVC requestSize且最小的一个PV：

```go
if smallestVolume == nil || smallestVolumeQty.Cmp(volumeQty) > 0 {
    smallestVolume = volume
    smallestVolumeQty = volumeQty
}
if smallestVolume != nil {
    // Found a matching volume
    return smallestVolume, nil
}
```

以上是PV和PVC的调度绑定流程。

#### Dynamic Provisioning

这个过程在PersistentVolumeController中完成，而当Pod在实际使用Volume前，需要通过Attach以及Mount流程后，才能真正进行使用。

而实际的应用场景则是，在环境中可能没有提前创建好可供“bound”的PV，这时候**Dynamic Provisioning**就派上用场了。

使用Dynamic Provisioning方式很简单，通过定义StorageClass就可以完成。

以Rook-Ceph的RBD服务为例，可以创建如下格式的StorageClass，以提供块存储服务：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block-service
  provisioner: ceph.rook.io/block
  parameters:
    pool: replicapool
    clusterNamespace: rook-ceph
```

通过在PVC中声明storageClassName字段，就可以进行动态使用了：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: block-service
  resources:
    requests:
      storage: 30Gi
```

在PVController watch到动态PVC被声明后，首先会寻找该PVC对应的plugin和storageClass:

```go
plugin, storageClass, err := ctrl.findProvisionablePlugin(claim)
```

这个过程会通过PersistentVolumeController的findProvisionablePlugin方法来进行寻找in-tree plugin，而find过程的关键在于通过PVC声明的storageClassName寻找对应的in-tree Plugin：

```go
// Find a plugin for the class
if ctrl.csiMigratedPluginManager.IsMigrationEnabledForPlugin(class.Provisioner) {
    // CSI migration scenario - do not depend on in-tree plugin
    return nil, class, nil
}
plugin, err := ctrl.volumePluginMgr.FindProvisionablePluginByName(class.Provisioner)
if err != nil {
    if !strings.HasPrefix(class.Provisioner, "kubernetes.io/") {
        // External provisioner is requested, do not report error
        return nil, class, nil
    }
    return nil, class, err
}
return plugin, class, nil
```

在1.14之后，PVController会先判断是否属于in-tree plugin到CSI的迁移(migration)场景，如果属于，则会将in-tree的plugin迁移到CSI，关于migration的产生背景，可以看下这篇介绍：https://kubernetes.io/blog/2019/12/09/kubernetes-1-17-feature-csi-migration-beta/

简单来说，为了支持Plugin机制的广泛使用，K8S社区越来越倾向于减少in-tree的代码，而通过Plugin的机制来进行扩展，原先in-tree的Plugin也被通过migration的机制，逐渐往CSI上迁，从中也能看出K8S社区对扩展性的考量，未来K8S极有可能成为Plugin的“媒介”系统（目前还未采用Plugin机制的，仅有kube-scheduler，而随着K8S社区的不断演进，kube-scheduler的默认调度器也会和CSI、CNI一样，支持自定义调度插件）。

继续往下分析，PVController会通过scheduleOperation来传入PV的Operation方法作为闭包，scheduleOperation的作用主要是通过grm（goroutinemap）的读写锁来判定，是否有Operation已经在运行中，运行中的作业会被预先加入goroutinemap中，用以判断。

```go
// goroutinemap
type goRoutineMap struct {
	operations                map[string]operation
	exponentialBackOffOnError bool
	cond                      *sync.Cond
	lock                      sync.RWMutex
}
```



#### Attach & Mount

在实际挂载时，通过ADController调用CSI的Attach操作，并在kubelet中调用Mount操作，完成存储卷和Pod的挂载过程。

在ADController中，首先会构建出PV对应的VolumeSpec，

```go
// NewSpecFromPersistentVolume creates an Spec from an v1.PersistentVolume
func NewSpecFromPersistentVolume(pv *v1.PersistentVolume, readOnly bool) *Spec {
	return &Spec{
		PersistentVolume: pv,
		ReadOnly:         readOnly,
	}
}
```

之后根据VolumeSpec寻找到plugin， 通过调用operation_executor，完成Attach操作。

```go
func (oe *operationExecutor) AttachVolume(
	volumeToAttach VolumeToAttach,
	actualStateOfWorld ActualStateOfWorldAttacherUpdater) error {
	generatedOperations :=
		oe.operationGenerator.GenerateAttachVolumeFunc(volumeToAttach, actualStateOfWorld)

	if util.IsMultiAttachAllowed(volumeToAttach.VolumeSpec) {
		return oe.pendingOperations.Run(
			volumeToAttach.VolumeName, "" /* podName */, volumeToAttach.NodeName, generatedOperations)
	}

	return oe.pendingOperations.Run(
		volumeToAttach.VolumeName, "" /* podName */, "" /* nodeName */, generatedOperations)
}
```

而Mount操作则在kubelet中进行，在kubelet中会生成VolumeManager对象。

关于VolumeManager的处理逻辑会在kubelet的详细介绍文章中介绍。

