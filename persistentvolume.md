有两种PV的提供方式:静态或者动态

Static
集群管理员创建多个PV。 它们携带可供集群用户使用的真实存储的详细信息。 它们存在于Kubernetes API中，可用于消费。
Dynamic
当管理员创建的静态PV都不匹配用户的PersistentVolumeClaim时，集群可能会尝试为PVC动态配置卷。 此配置基于StorageClasses：PVC必须请求一个类，并且管理员必须已创建并配置该类才能进行动态配置。 要求该类的声明有效地为自己禁用动态配置

## PV的生命周期：
PV是集群中的资源。 PVC是对这些资源的请求，也是对资源的索赔检查。 PV和PVC之间的相互作用遵循这个生命周期:

Provisioning ——-> Binding ——–>Using——>Releasing——>Recycling

Provisioning
有两种PV的提供方式:静态或者动态

Static
集群管理员创建多个PV。 它们携带可供集群用户使用的真实存储的详细信息。 它们存在于Kubernetes API中，可用于消费。
Dynamic
当管理员创建的静态PV都不匹配用户的PersistentVolumeClaim时，集群可能会尝试为PVC动态配置卷。 此配置基于StorageClasses：PVC必须请求一个类，并且管理员必须已创建并配置该类才能进行动态配置。 要求该类的声明有效地为自己禁用动态配置

Binding
在动态配置的情况下，用户创建或已经创建了具有特定数量的存储请求和特定访问模式的PersistentVolumeClaim。 主机中的控制回路监视新的PVC，找到匹配的PV（如果可能），并将它们绑定在一起。 如果为新的PVC动态配置PV，则循环将始终将该PV绑定到PVC。 否则，用户总是至少得到他们要求的内容，但是卷可能超出了要求。 一旦绑定，PersistentVolumeClaim绑定是排他的，不管用于绑定它们的模式。

如果匹配的卷不存在，PVC将保持无限期。 随着匹配卷变得可用，PVC将被绑定。 例如，提供许多50Gi PV的集群将不匹配要求100Gi的PVC。 当集群中添加100Gi PV时，可以绑定PVC。

Using
Pod使用PVC作为卷。 集群检查声明以找到绑定的卷并挂载该卷的卷。 对于支持多种访问模式的卷，用户在将其声明用作pod中的卷时指定所需的模式。

一旦用户有声明并且该声明被绑定，绑定的PV属于用户，只要他们需要它。 用户通过在其Pod的卷块中包含persistentVolumeClaim来安排Pods并访问其声明的PV。

Releasing
当用户完成卷时，他们可以从允许资源回收的API中删除PVC对象。 当声明被删除时，卷被认为是“释放的”，但是它还不能用于另一个声明。 以前的索赔人的数据仍然保留在必须根据政策处理的卷上.

Reclaiming 
PersistentVolume的回收策略告诉集群在释放其声明后，该卷应该如何处理。 目前，卷可以是保留，回收或删除。 保留可以手动回收资源。 对于那些支持它的卷插件，删除将从Kubernetes中删除PersistentVolume对象，以及删除外部基础架构（如AWS EBS，GCE PD，Azure Disk或Cinder卷）中关联的存储资产。 动态配置的卷始终被删除


pv 支持的类型
```
GCEPersistentDisk 
AWSElasticBlockStore 
AzureFile 
AzureDisk 
FC (Fibre Channel) 
FlexVolume 
Flocker 
NFS 
iSCSI 
RBD (Ceph Block Device) 
CephFS 
Cinder (OpenStack block storage) 
Glusterfs 
VsphereVolume 
Quobyte Volumes 
HostPath (single node testing only – local storage is not supported in any way and WILL NOT WORK in a multi-node cluster) 
VMware Photon 
Portworx Volumes 
ScaleIO Volumes 
```

PV示例：
```
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv0003
  spec:
    capacity:
      storage: 5Gi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    storageClassName: slow
    nfs:
      path: /tmp
      server: 172.17.0.2
```

Capacity
通常，PV将具有特定的存储容量。 这是使用PV的容量属性设置的。 看到库伯纳斯资源模型，以了解容量预期的单位。

目前，存储大小是唯一可以设置或请求的资源。 未来的属性可能包括IOPS，吞吐量等

Access Modes
PersistentVolume可以以资源提供者支持的任何方式安装在主机上。 如下表所示，提供商将具有不同的功能，每个PV的访问模式都被设置为该特定卷支持的特定模式。 例如，NFS可以支持多个读/写客户端，但是特定的NFS PV可能会以只读方式在服务器上导出。 每个PV都有自己的一组访问模式来描述具体的PV功能。 
访问模式: 
ReadWriteOnce – the volume can be mounted as read-write by a single node (单node的读写) 
ReadOnlyMany – the volume can be mounted read-only by many nodes (多node的只读) 
ReadWriteMany – the volume can be mounted as read-write by many nodes (多node的读写) 
Notice:单个PV挂载的时候只支持一种访问模式 
PV提供插件支持的access mode参考kubernetes官方文档

Class
PV可以有一个类，通过将storageClassName属性设置为StorageClass的名称来指定。 特定类的PV只能绑定到请求该类的PVC。 没有storageClassName的PV没有类，只能绑定到不需要特定类的PVC。 
在过去，使用了注释volume.beta.kubernetes.io/storage-class而不是storageClassName属性。 该注释仍然可以工作，但将来Kubernetes版本将不再适用。

Reclaim Policy
目前的回收政策是：

Retain – manual reclamation 
Recycle – basic scrub (“rm -rf /thevolume/*”) 
Delete – associated storage asset such as AWS EBS, GCE PD, Azure Disk, or OpenStack Cinder volume is deleted

目前，只有NFS和HostPath支持回收。 AWS EBS，GCE PD，Azure Disk和Cinder卷支持删除

PV的状态
```
Available – a free resource that is not yet bound to a claim 
Bound – the volume is bound to a claim 
Released – the claim has been deleted, but the resource is not yet reclaimed by the cluster 
Failed – the volume has failed its automatic reclamation 
```

Mount Options
Kubernetes管理员可以指定在一个节点上挂载一个持久卷时的其他安装选项。 
您可以通过使用持久卷上的注释volume.beta.kubernetes.io/mount-options来指定安装选项。
```
apiVersion: "v1"
kind: "PersistentVolume"
metadata:
  name: gce-disk-1
  annotations:
    volume.beta.kubernetes.io/mount-options: "discard"
spec:
  capacity:
    storage: "10Gi"
  accessModes:
    - "ReadWriteOnce"
  gcePersistentDisk:
    fsType: "ext4"
    pdName: "gce-disk-1
```