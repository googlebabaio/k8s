<!-- toc -->
Docker提供了Volumes，Volume 是磁盘上的文件夹并且没有生命周期的管理。
Kubernetes 中的 Volume 是存储的抽象，并且能够为Pod提供多种存储解决方案。
Volume 最终会映射为Pod中容器可访问的一个文件夹或裸设备，但是背后的实现方式可以有很多种。

## volume的种类：
* cephfs
* configMap
* emptyDir
* hostPath
* local
* nfs
* persistentVolumeClaim

### emptyDir
emptyDir在Pod被分配到Node上之后创建，它的生命周期和所属的 Pod 完全一致，emptyDir主要作用可以在同一 Pod 内的不同容器之间共享工作过程中产生的文件。初始的时候为一个空文件夹，当Pod从Node中移除时，emptyDir将被永久删除。容器挂掉并不会导致emptyDir被删除，但是如果Pod从Node上被删除（Pod被删除，或者Pod发生迁移），emptyDir也会被删除，并且永久丢失。。emptyDir适用于一些临时存放数据的场景。
示例：
```
# cat hostpath.yaml
apiVersion: v1
kind: Pod
metadata:
   name: busybox
spec:
  containers:
   - name : busybox
     image: busybox
     imagePullPolicy: IfNotPresent
     command:
      - sleep
      - "3600"
     volumeMounts:
      - mountPath: /busybox-data
        name: data
  volumes:
   - name: data
     emptyDir: {}
```

### Hostpath
hostPath就是将Node节点的文件系统挂载到Pod中。但是如果 Pod 发生跨主机的重建，其内容就很难保证了。这种卷一般和DaemonSet搭配使用。hostPath允许挂载Node上的文件系统到Pod里面去。如果Pod有需要使用Node上的东西，可以使用hostPath，不过不过建议使用，因为理论上Pod不应该感知Node的。
示例：
```
# cat hostpath.yaml
apiVersion: v1
kind: Pod
metadata:
   name: busybox
spec:
  containers:
   - name : busybox
     image: busybox
     imagePullPolicy: IfNotPresent
     command:
      - sleep
      - "3600"
     volumeMounts:
      - mountPath: /busybox-data
        name: data
  volumes:
   - hostPath:
      path: /tmp
     name: data
```

emptyDir和hostPat很多场景是无法满足持久化需求，因为在Pod发生迁移的时候，数据都无法进行转移的，这就需要分布式文件系统的支持。
