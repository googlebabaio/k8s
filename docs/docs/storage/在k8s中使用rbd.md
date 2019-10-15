<!-- toc -->
# 在k8s中使用rbd

## 存储可以分为三类
- 对象存储（例如aws s3、aliyun oss）
- 文件存储（例如nfs、nas）
- 块存储。（例如Ceph RBD

## ceph集群使用ceph rbd
对于k8s来说，需要的Ceph信息有这些：
- IP地址
- 端口号
- 管理员用户名
- 管理员keyring

node上准备ceph
```
yum install ceph-common ceph-fs-common -y
```

ceph命令行需要2个配置文件，具体配置问ceph的管理团队
```
/etc/ceph/ceph.conf
/etc/ceph/ceph.client.admin.keyring
```

## rbd使用过程
RBD是这样使用的：用户在Ceph上创建Pool（逻辑隔离），然后在Pool中创建image（实际存储介质），之后再将image挂载到本地服务器的某个目录上。
```
# rbd list    #列出默认pool下的image
# rbd list -p k8s   #列出pool k8s下的image
# rbd create foo -s 1024  #在默认pool中创建名为foo的image，大小为1024MB
# rbd map foo #将ceph集群的image映射到本地的块设备
/dev/rbd0
# ls -l /dev/rbd0   #是b类型
brw-rw---- 1 root disk 252, 0 May 22 20:57 /dev/rbd0
$ rbd showmapped    #查看已经map的rbd image
id pool image snap device
0  rbd  foo   -    /dev/rbd0
# mount /dev/rbd1 /mnt/bar/   #此时去mount会失败，因为image还没有格式化文件系统
mount: /dev/rbd1 is write-protected, mounting read-only
mount: wrong fs type, bad option, bad superblock on /dev/rbd1,
       missing codepage or helper program, or other error

       In some cases useful info is found in syslog - try
       dmesg | tail or so.
# mkfs.ext4 /dev/rbd0   #格式化为ext4
...
Writing superblocks and filesystem accounting information: done
# mount /dev/rbd0 /mnt/foo/   #重新挂载
# df -h |grep foo             #ok
/dev/rbd0       976M  2.6M  907M   1% /mnt/foo
```

上面都是在默认pool中做的，在k8s里，建议使用自己的pool了。
```
# ceph osd lspools # 看看已经有那些pool
# ceph osd pool create k8s 128 #创建pg_num 为128的名为k8s的pool
# rados df
# rbd create foobar -s 1024 -p k8s #在k8s pool中创建名为foobar的image，大小为1024MB
```

## 直接使用RBD作为volume
前面已经创建好了image foobar，给volume用简单直接粗暴，挂就行了。

```
apiVersion: v1
kind: Pod
metadata:
  name: rbd
spec:
  containers:
    - image: gcr.io/nginx
      name: rbd-rw
      volumeMounts:
      - name: rbdpd
        mountPath: /mnt/rbd
  volumes:
    - name: rbdpd
      rbd:
        monitors:
        - '1.2.3.4:6789'
        pool: k8s
        image: foobar
        fsType: ext4
        readOnly: false
        user: admin
        keyring: /etc/ceph/ceph.client.admin.keyring
```

Pod启动后，可以看到文件系统由k8s做好并挂载到了容器里。我们将/etc/hosts文件拷贝到/mnt/rbd/目录去。
```
# kubectl exec rbd -- df -h|grep rbd
/dev/rbd6       976M  2.6M  907M   1% /mnt/rbd
# kubectl exec rbd -- cp /etc/hosts /mnt/rbd/
# kubectl exec rbd -- cat /mnt/rbd/hosts
# Kubernetes-managed hosts file.
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
fe00::0 ip6-mcastprefix
fe00::1 ip6-allnodes
fe00::2 ip6-allrouters
10.244.3.249    rbd
```

然后将Pod删除、重新挂载foobar image。
前面Pod要求各node上都要有keyring文件，很不方便也不安全。新的Pod我使用推荐的做法：secret（虽然也安全不到哪里）

先创建一个secret
```
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
type: "kubernetes.io/rbd"
data:
  key: QVFCXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX9PQ==
```

在新的Pod里Ref这个secret
```
apiVersion: v1
kind: Pod
metadata:
  name: rbd3
spec:
  containers:
    - image: gcr.io/nginx
      name: rbd-rw
      volumeMounts:
      - name: rbdpd
        mountPath: /mnt/rbd
  volumes:
    - name: rbdpd
      rbd:
        monitors:
        - '1.2.3.4:6789'
        pool: k8s
        image: foobar
        fsType: ext4
        readOnly: false
        user: admin
        secretRef:
          name: ceph-secret
```


## RBD用作PV/PVC
先在k8s pool里创建一个名为pv的 image。
```
rbd image create pv -s 1024 -p k8s
```

再创建一个PV，使用上面创建的image pv。
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ceph-rbd-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
      - '1.2.3.4:6789'
    pool: k8s
    image: pv
    user: admin
    secretRef:
      name: ceph-secret
    fsType: ext4
    readOnly: false
  persistentVolumeReclaimPolicy: Recycle
```

创建一个PVC，要求一块1G的存储。
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-rbd-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```


## 利用storage class使用RBD
先创建一个SC。根PV一样，SC也是集群范围的（RBD认为是fast的）。
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: fast
provisioner: kubernetes.io/rbd
parameters:
  monitors: 172.25.60.3:6789
  adminId: admin
  adminSecretName: ceph-secret
  adminSecretNamespace: resource-quota
  pool: k8s
  userId: admin
  userSecretName: ceph-secret
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
```

之后创建应用的时候，需要同时创建 pvc+pod，二者通过claimName关联。pvc中需要指定其storageClassName为上面创建的sc的name（即fast）。
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: rbd-pvc-pod-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: fast
```

RBD只支持 ReadWriteOnce 和 ReadOnlyAll，不支持ReadWriteAll。注意这两者的区别点是，不同nodes之间是否可以同时挂载。同一个node上，即使是ReadWriteOnce，也可以同时挂载到2个容器上的。
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: rbd-pvc-pod
  name: ceph-rbd-sc-pod2
spec:
  containers:
  - name: ceph-rbd-sc-nginx
    image: gcr.io/nginx
    volumeMounts:
    - name: ceph-rbd-vol1
      mountPath: /mnt/ceph-rbd-pvc/nginx
      readOnly: false
  volumes:
  - name: ceph-rbd-vol1
    persistentVolumeClaim:
      claimName: rbd-pvc-pod-pvc
```

再来一个statefulset的例子:
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: gcr.io/nginx
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "fast"
      resources:
        requests:
          storage: 8Gi
```


## 参考:
[使用Ceph RBD为Kubernetes集群提供存储卷](https://tonybai.com/2016/11/07/integrate-kubernetes-with-ceph-rbd/)
[初试 Kubernetes 集群使用 Ceph RBD 块存储](https://blog.csdn.net/aixiaoyang168/article/details/78999851)
