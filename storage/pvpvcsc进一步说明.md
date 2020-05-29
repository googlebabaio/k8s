<!-- toc -->
# 再说说 pv、pvc和storageclass的相关概念

##  三个概念

- pv ：持久化卷， 支持本地存储和网络存储， 例如hostpath，ceph rbd， nfs等，只支持两个属性， capacity和accessModes。其中capacity只支持size的定义，不支持iops等参数的设定，accessModes有三种，ReadWriteOnce（被单个node读写）， ReadOnlyMany（被多个nodes读）， ReadWriteMany（被多个nodes读写）
- storageclass：另外一种提供存储资源的方式， 提供更多的层级选型， 如iops等参数。 但是具体的参数与提供方是绑定的。 如aws和gce它们提供的storageclass的参数可选项是有不同的。
- pvc ： 对pv或者storageclass资源的请求， pvc 对 pv 类比于pod 对不同的cpu， mem的请求。

## 六个生命周期
- Provisioning
- Binding
- Using
- Releasing
- Reclaiming
- Recycling

k8s对pv和pvc之间的交互的生命周期进行管理：
- provisioning－  配置阶段， 分为static， dynamic两种方式。静态的方式是创建一系列的pv，然后pvc从pv中请求。 动态的方式是基于storageclass的。

- Binding － 绑定阶段， pvc根据请求的条件筛选并绑定对应的pv。 一定pvc绑定pv后， 就会排斥其它绑定，即其它pvc无法再绑定同一个pv，即使这个pv设定的access mode允许多个node读写。 此外 ，pvc 如何匹配不到相应条件的pv， 那么就会显示unbound状态， 直到匹配为止。 需要注意的是，pvc请求100Gi大小的存储，即使用户创建了很多50Gi大小的存储， 也是无法被匹配的。

- Using－ 使用阶段， pods 挂载存储， 即在pod的template文件中定义volumn使用某个pvc。

- Releasing － 释放阶段， 当pvc对象被删除后， 就处于释放阶段。 在这个阶段， 使用的pv还不能被其它的pvc请求。 之前数据可能还会留存下来， 取决于用户在pv中设定的policy， 见persistentVolumeReclaimPolicy。

- Reclaiming － 重声明阶段。 到这个阶段， 会告诉cluster如何处理释放的pv。 数据可能被保留（需要手工清除）， 回收和删除。动态分配的存储总是会被删除掉的。

- Recycling － 回收阶段。回收阶段会执行基本的递归删除（取决于volumn plugins的支持），把pv上的数据删除掉， 以使pv可以被新的pvc请求。 用户也可以自定义一个 recycler pod ， 对数据进行删除。

## 三种PV的访问模式

- ReadWriteOnce：是最基本的方式，可读可写，但只支持被单个Pod挂载。
- ReadOnlyMany：可以以只读的方式被多个Pod挂载。
- ReadWriteMany：这种存储可以以读写的方式被多个Pod共享。不是每一种存储都支持这三种方式，像共享方式，目前支持的还比较少，比较常用的是NFS。在PVC绑定PV时通常根据两个条件来绑定，一个是存储的大小，另一个就是访问模式。

## 九个PV Plugins

pv是以plugin的形式来提供支持的， 考虑到私有云的使用场景， 排除掉azure， aws，gce等公有云厂商绑定的plugin， 有9个插件值得关注。这些plugin所支持的accessmode是不同的。  分别是：

存储Plugin   |   ReadWriteOnce   |   ReadOnlyMany   |   ReadWriteMany   |   备注
--|---|---|---|--
FC (Fibre Channel)   |   支持   |   支持   |   不支持   |
NFS   |   支持   |   支持   |   支持   |
iSCSI   |   支持   |   支持   |   不支持   |
RBD (Ceph Block Device)   |   支持   |   支持   |   不支持   |
CephFS   |   支持   |   支持   |   支持   |
Cinder (OpenStack block storage)   |   支持   |   不支持   |   不支持   |
Glusterfs   |   支持   |   支持   |   支持   |
VsphereVolume   |   支持   |   不支持   |   不支持   |
HostPath   |   支持   |   不支持   |   不支持   |   只支持单节点， 不支持跨节点



## 三个重声明策略(reclaim policy)

- Retain –  手动重新使用
- Recycle – 基本的数据擦除 (“rm -rf /thevolume/*”)
- Delete – 相关联的后端存储卷删除， 后端存储比如AWS EBS, GCE PD, Azure Disk, or OpenStack Cinder


需要特别注意的是只有本地盘和nfs支持数据盘Recycle 擦除回收， AWS EBS, GCE PD, Azure Disk, and Cinder 存储卷支持Delete策略


##  四个阶段(volumn  phase)

一个存储卷会处于下面几个阶段中的一个阶段：
- Available –资源可用， 还没有被声明绑定
- Bound –  被声明绑定
- Released – 绑定的声明被删除了，但是还没有被集群重声明
- Failed – 自动回收失败

##  四个PV选择器
在PVC中绑定一个PV，可以根据下面几种条件组合选择
- Access Modes， 按照访问模式选择pv
- Resources， 按照资源属性选择， 比如说请求存储大小为8个G的pv
- Selector， 按照pv的label选择
- Class， 根据StorageClass的class名称选择, 通过annotation指定了Storage Class的名字, 来绑定特定类型的后端存储

关于根据class过滤出pv的说明：
> 所有的 PVC 都可以在不使用 StorageClass 注解的情况下，直接使用某个动态存储。把一个StorageClass 对象标记为 “default” 就可以了。StorageClass 用注解storageclass.beta.kubernetes.io/is-default-class 就可以成为缺省存储。
有了缺省的 StorageClass，用户创建 PVC 就不用 storage-class 的注解了，1.4 中新加入的DefaultStorageClass 准入控制器会自动把这个标注指向缺省存储类。
>  PVC 指定特定storageClassName，如fast时， 绑定名称为fast的storageClass
>  PVC中指定storageClassName为“”时， 绑定no class的pv（pv中无class annotation， 或者其值为“”）
> PVC不指定storageClassName时， DefaultStorageClass admission plugin 开启与否（在apiserver启动时可以指定）， 对default class的解析行为是不同的。
当DefaultStorageClass admission plugin启用时， 针对没有storageClass annotation的pvc，DefaultStorageClass会分配一个默认的class， 这个默认的class需要用户指定，比如在创建storageclass对象时加入annotation,如  storageclass.beta.kubernetes.io/is-default-class:  “true” 。如果有多个默认的class， 则pvc会被拒绝创建， 如果用户没有指定默认的class， 则这个DefaultStorageClass admission plugin不会起任何作用。 pvc会找那些no class的pv做绑定。
当DefaultStorageClass admission plugin没有启用时， 针对没有storageClass annotation的pvc， 会绑定no class的pv（pv中无class annotation， 或者其值为“”）



## 五个可移植性建议
1. 把 pvc 和 其它一系列配置放一起， 比如说deployment，configmap
2. 不要把pv放在其它配置里， 因为用户可能没有权限创建pv
3. 初始化 pvc 模版的时候， 提供一个storageclass
4. 在工具软件中，watch那些没有bound的pvc，并呈现给用户
5. 集群启动的时候启用DefaultStorageClass， 但是不要指定某一类特定的class， 因为不同provisioner的class，参数很难一致


# 如何创建和使用PV， PVC
## pv示例
```yml
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

## pvc示例
```yml
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

## 在pod中使用pvc
```yml
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


# 参考文献
- [官方文档之持久化存储](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [官方文档之存储方案](http://kubernetes.io/docs/user-guide/volumes/)
- [flocker get started guide](https://clusterhq.com/flocker/getting-started/)
- [k8s-flocker-example](https://github.com/kubernetes/kubernetes/tree/master/examples/volumes/flocker)
- [Docker容器的持久存储模式介绍](http://www.dockerinfo.net/1000.html)
- [开源数据卷管理工具flocker](http://dockone.io/article/1549)
- [flcoker 支持的后端列表](https://docs.clusterhq.com/en/latest/supported/index.html)
- [Docker容器对存储的定义（Volume 与 Volume Plugin）](http://dockone.io/article/1257)
- [典型容器存储项目揭密：Flocker，Portworx和VSAN](http://mp.weixin.qq.com/s?__biz=MzAwNzUyNzI5Mw==&mid=2730790315&idx=1&sn=01fff8ee701b99dbe64eb004e0a1cb46&scene=1&srcid=0919gZliDxa3HMA8vptYv5LX#rd)
- [官方文档之持久化存储](http://kubernetes.io/docs/user-guide/persistent-volumes/)
- [探讨容器中使用块存储](https://www.kubernetes.org.cn/914.html)
- [Configuring a Pod to Use a PersistentVolume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)
- [Rook存储：Kubernetes中最优秀的存储](http://www.dockone.io/article/2156)
- [基于docker部署ceph以及修改docker image](http://www.dockerinfo.net/4440.html)
