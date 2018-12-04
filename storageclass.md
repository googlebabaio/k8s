每个StorageClass包含字段provisioninger和参数，当属于类的PersistentVolume需要动态配置时使用。

StorageClass对象的名称很重要，用户可以如何请求特定的类。 管理员在首次创建StorageClass对象时设置类的名称和其他参数，并且在创建对象后无法更新对象。

管理员可以仅为不要求任何特定类绑定的PVC指定默认的StorageClass：有关详细信息，请参阅PersistentVolumeClaim部分。


```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```

Provisioner
存储类有一个供应商，它确定用于配置PV的卷插件。 必须指定此字段。

您不限于指定此处列出的“内部”供应商（其名称前缀为“kubernetes.io”并与Kubernetes一起运送）。 您还可以运行和指定外部提供程序，它们是遵循Kubernetes定义的规范的独立程序。 外部提供者的作者对代码的生命周期，供应商的运输状况，运行状况以及使用的卷插件（包括Flex）等都有充分的自主权。存储库kubernetes-incubator /外部存储库包含一个库 用于编写实施大部分规范的外部提供者以及各种社区维护的外部提供者。

Parameters
存储类具有描述属于存储类的卷的参数。 取决于供应商，可以接受不同的参数。 例如，参数类型的值io1和参数iopsPerGB特定于EBS。 当省略参数时，使用一些默认值。

AWS/GCE/Glusterfs/

OpenStack Cinder
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

其他配置参考：
https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes

建议：
如果您正在编写在各种群集上运行并需要持久存储的配置模板或示例，我们建议您使用以下模式：

在您的配置文件夹（包括部署，ConfigMaps等）中包含PersistentVolumeClaim对象。 
在配置中不要包含PersistentVolume对象，因为实例化配置的用户可能没有创建PersistentVolumes的权限。 
给用户提供实例化模板时提供存储类名称的选项。 
１．　如果用户提供存储类名称，并且集群是1.4或更高版本，请将该值放入PVC的volume.beta.kubernetes.io/storage-class注释中。如果集群的管理员启用了StorageClasses，这将导致PVC与正确的存储类匹配。 
２．　如果用户不提供存储类名称或者集群是版本1.3，那么在PVC上放置一个volume.alpha.kubernetes.io/storage-class：default注释。 
这将导致在某些群集上为用户自动配置PV，并具有合理的默认特性。 
尽管在名称中使用了alpha，但这个注释背后的代码具有beta级别的支持。 
３．　不要使用volume.beta.kubernetes.io/storage-class：包含空字符串的任何值，因为它将阻止DefaultStorageClass接纳控制器运行（如果启用）。

在您的工具中，请注意在一段时间后未绑定的PVC，并将其表示给用户，因为这可能表明集群没有动态存储支持（在这种情况下，用户应创建匹配的PV）或集群没有存储系统（在这种情况下，用户无法部署需要PVC的配置）。

在将来，我们预计大多数集群都将启用DefaultStorageClass，并提供某种形式的存储。但是，可能没有任何存储类名可用于所有集群，因此默认情况下继续不设置。在某种程度上，alpha注释将不再有意义，但PVC上的未设置的storageClass字段将具有所需的效果。