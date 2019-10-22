<!-- toc -->
# PV
## PV的提供方式
- 静态
- 动态

## PV的生命周期：
PV是集群中的资源。 PVC是对这些资源的请求，也是对资源的索赔检查。 PV和PVC之间的相互作用遵循这个生命周期:
```
Provisioning ——-> Binding ——–> Using ——> Releasing ——> Recycling
```
### Static
集群管理员创建多个PV。 它们携带可供集群用户使用的真实存储的详细信息。 它们存在于Kubernetes API中，可用于消费。
## Dynamic
当管理员创建的静态PV都不匹配用户的PersistentVolumeClaim时，集群可能会尝试为PVC动态配置卷。 此配置基于StorageClasses：PVC必须请求一个类，并且管理员必须已创建并配置该类才能进行动态配置。 要求该类的声明有效地为自己禁用动态配置

### Binding
在动态配置的情况下，用户创建或已经创建了具有特定数量的存储请求和特定访问模式的PersistentVolumeClaim。 主机中的控制回路监视新的PVC，找到匹配的PV（如果可能），并将它们绑定在一起。 如果为新的PVC动态配置PV，则循环将始终将该PV绑定到PVC。 否则，用户总是至少得到他们要求的内容，但是卷可能超出了要求。 一旦绑定，PersistentVolumeClaim绑定是排他的，不管用于绑定它们的模式。

如果匹配的卷不存在，PVC将保持无限期。 随着匹配卷变得可用，PVC将被绑定。 例如，提供许多50Gi PV的集群将不匹配要求100Gi的PVC。 当集群中添加100Gi PV时，可以绑定PVC。

### Using
Pod使用PVC作为卷。 集群检查声明以找到绑定的卷并挂载该卷的卷。 对于支持多种访问模式的卷，用户在将其声明用作pod中的卷时指定所需的模式。

一旦用户有声明并且该声明被绑定，绑定的PV属于用户，只要他们需要它。 用户通过在其Pod的卷块中包含persistentVolumeClaim来安排Pods并访问其声明的PV。

### Releasing
当用户完成卷时，他们可以从允许资源回收的API中删除PVC对象。 当声明被删除时，卷被认为是“释放的”，但是它还不能用于另一个声明。 以前的索赔人的数据仍然保留在必须根据政策处理的卷上.

### Reclaiming
PersistentVolume的回收策略告诉集群在释放其声明后，该卷应该如何处理。 目前，卷可以是保留，回收或删除。 保留可以手动回收资源。 对于那些支持它的卷插件，删除将从Kubernetes中删除PersistentVolume对象，以及删除外部基础架构（如AWS EBS，GCE PD，Azure Disk或Cinder卷）中关联的存储资产。 动态配置的卷始终被删除


## pv 支持的类型
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

## PV示例：
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

## PV的存储容量--Capacity
通常，PV将具有特定的存储容量。 这是使用PV的容量属性设置的。 看到库伯纳斯资源模型，以了解容量预期的单位。

目前，存储大小是唯一可以设置或请求的资源。 未来的属性可能包括IOPS，吞吐量等

## PV的访问模式
PersistentVolume可以以资源提供者支持的任何方式安装在主机上。 提供商将具有不同的功能，每个PV的访问模式都被设置为该特定卷支持的特定模式。
 例如，NFS可以支持多个读/写客户端，但是特定的NFS PV可能会以只读方式在服务器上导出。 每个PV都有自己的一组访问模式来描述具体的PV功能。

访问模式:
- ReadWriteOnce – the volume can be mounted as read-write by a single node (单node的读写)
- ReadOnlyMany – the volume can be mounted read-only by many nodes (多node的只读)
- ReadWriteMany – the volume can be mounted as read-write by many nodes (多node的读写)
- Notice:单个PV挂载的时候只支持一种访问模式
- PV提供插件支持的access mode参考kubernetes官方文档

## Class
PV可以有一个类，通过将storageClassName属性设置为StorageClass的名称来指定。 特定类的PV只能绑定到请求该类的PVC。 没有storageClassName的PV没有类，只能绑定到不需要特定类的PVC。
在过去，使用了注释volume.beta.kubernetes.io/storage-class而不是storageClassName属性。 该注释仍然可以工作，但将来Kubernetes版本将不再适用。

## PV的回收策略
目前的回收政策是：

- Retain – manual reclamation
- Recycle – basic scrub (“rm -rf /thevolume/*”)
- Delete – associated storage asset such as AWS EBS, GCE PD, Azure Disk, or OpenStack Cinder volume is deleted

目前，只有NFS和HostPath支持回收。 AWS EBS，GCE PD，Azure Disk和Cinder卷支持删除

## PV的状态
```
Available – a free resource that is not yet bound to a claim
Bound – the volume is bound to a claim
Released – the claim has been deleted, but the resource is not yet reclaimed by the cluster
Failed – the volume has failed its automatic reclamation
```

## Mount Options
Kubernetes管理员可以指定在一个节点上挂载一个持久卷时的其他安装选项。
可以通过使用持久卷上的注释volume.beta.kubernetes.io/mount-options来指定安装选项。
示例:
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


# PVC

每个PVC包含spec和status !

yaml示例
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

## Access Modes
当请求具有特定访问模式的存储时，声明使用与卷相同的约定

## Resources
声明（如pod）可以请求特定数量的资源。 在这种情况下，请求用于存储。 相同的资源模型适用于卷和声明

## Selector
声明可以指定标签选择器以进一步过滤该卷集。 只有标签与选择器匹配的卷才能绑定到声明。 选择器可以由两个字段组成：

## matchLabels - 卷必须具有带此值的标签
matchExpressions - 通过指定关键字和值的关键字，值列表和运算符所做的要求列表。 有效运算符包括In，NotIn，Exists和DoesNotExist。
所有来自matchLabels和matchExpressions的要求都与AND一起使用，所有这些要求都必须满足才能匹配。

## Class
声明可以通过使用属性storageClassName指定StorageClass的名称来请求特定的类。只有所请求的类的PV，与PVC相同的storageClassName的PV可以绑定到PVC。

PVC不一定要求一个className。它的storageClassName设置为等于“”的PVC总是被解释为请求没有类的PV，因此它只能绑定到没有类的PV（没有注释或一个等于“”）。没有storageClassName的PVC不完全相同，并且根据是否启用了DefaultStorageClass入门插件，集群的处理方式不同。

如果接纳插件已打开，则管理员可以指定默认的StorageClass。没有storageClassName的所有PVC只能绑定到该默认的PV。通过将StorageClass对象中的`annotation storageclass.kubernetes.io/is-default-class`设置为“true”来指定默认的StorageClass。如果管理员没有指定默认值，则集群会对PVC创建做出响应，就像入门插件被关闭一样。如果指定了多个默认值，则验收插件禁止创建所有PVC。
如果入门插件已关闭，则不存在默认StorageClass的概念。没有storageClassName的所有PVC只能绑定到没有类的PV。在这种情况下，没有storageClassName的PVC的处理方式与将其storageClassName设置为“”的PVC相同。

根据安装方法，安装过程中可以通过addon manager在Kubernetes群集中部署默认的StorageClass。

当PVC指定一个选择器，除了请求一个StorageClass之外，这些要求被AND组合在一起：只有所请求的类和所请求的标签的PV可能被绑定到PVC。请注意，当前，具有非空选择器的PVC不能为其动态配置PV。

在过去，使用了注释volume.beta.kubernetes.io/storage-class，而不是storageClassName属性。该注释仍然可以工作，但在未来的Kubernetes版本中它将不被支持。


## 声明PVC作为Volumes
Pods通过将声明用作卷来访问存储。 声明必须存在于与使用声明的pod相同的命名空间中。 群集在pod的命名空间中查找声明，并使用它来获取支持声明的PersistentVolume。 然后将体积安装到主机并进入Pod。
```
kind: Pod
apiVersion: v1
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: dockerfile/nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```
PersistentVolumes绑定是独占的，并且由于PersistentVolumeClaims是命名空间对象，所以只能在一个命名空间内安装“许多”模式（ROX，RWX）—PVC支持被多个pod挂载

# StorageClass
每个 StorageClass 包含字段provisioninger和参数，当属于类的PersistentVolume需要动态配置时使用。

StorageClass对象的名称很重要，用户可以如何请求特定的类。 管理员在首次创建StorageClass对象时设置类的名称和其他参数，并且在创建对象后无法更新对象。

管理员可以仅为不要求任何特定类绑定的PVC指定默认的StorageClass：有关详细信息，请参阅PersistentVolumeClaim部分。

示例:
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```

## Provisioner
存储类有一个供应商，它确定用于配置PV的卷插件。 必须指定此字段。

您不限于指定此处列出的“内部”供应商（其名称前缀为“kubernetes.io”并与Kubernetes一起运送）。 您还可以运行和指定外部提供程序，它们是遵循Kubernetes定义的规范的独立程序。 外部提供者的作者对代码的生命周期，供应商的运输状况，运行状况以及使用的卷插件（包括Flex）等都有充分的自主权。存储库kubernetes-incubator /外部存储库包含一个库 用于编写实施大部分规范的外部提供者以及各种社区维护的外部提供者。

## Parameters
存储类具有描述属于存储类的卷的参数。 取决于供应商，可以接受不同的参数。 例如，参数类型的值io1和参数iopsPerGB特定于EBS。 当省略参数时，使用一些默认值。

示例:
AWS/GCE/Glusterfs/OpenStack Cinder
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gold
provisioner: kubernetes.io/cinder
parameters:
  type: fast
  availability: nova
```

> 其他配置参考：
https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes

# 使用建议
- 如果正在编写在各种群集上运行并需要持久存储的配置模板或示例，建议使用以下模式：
- 在配置文件夹（包括部署，ConfigMaps等）中包含PersistentVolumeClaim对象。
- 在配置中不要包含PersistentVolume对象，因为实例化配置的用户可能没有创建PersistentVolumes的权限。
- 给用户提供实例化模板时提供存储类名称的选项。
  - 如果用户提供存储类名称，并且集群是1.4或更高版本，请将该值放入PVC的`volume.beta.kubernetes.io/storage-class`注释中。如果集群的管理员启用了StorageClasses，这将导致PVC与正确的存储类匹配。
  - 如果用户不提供存储类名称或者集群是版本1.3，那么在PVC上放置一个`volume.alpha.kubernetes.io/storage-class`：default注释。这将导致在某些群集上为用户自动配置PV，并具有合理的默认特性。
  - 尽管在名称中使用了alpha，但这个注释背后的代码具有beta级别的支持。
  - 不要使用`volume.beta.kubernetes.io/storage-class`：包含空字符串的任何值，因为它将阻止DefaultStorageClass接纳控制器运行（如果启用）。
- 在我们的工具中，请注意在一段时间后未绑定的PVC，并将其表示给用户，因为这可能表明集群没有动态存储支持（在这种情况下，用户应创建匹配的PV）或集群没有存储系统（在这种情况下，用户无法部署需要PVC的配置）。
- 在将来，预计大多数集群都将启用DefaultStorageClass，并提供某种形式的存储。但是，可能没有任何存储类名可用于所有集群，因此默认情况下继续不设置。在某种程度上，alpha注释将不再有意义，但PVC上的未设置的storageClass字段将具有所需的效果。
