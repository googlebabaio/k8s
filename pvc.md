每个PVC包含spec和status

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
Access Modes
当请求具有特定访问模式的存储时，声明使用与卷相同的约定

Resources
声明（如pod）可以请求特定数量的资源。 在这种情况下，请求用于存储。 相同的资源模型适用于卷和声明

Selector
声明可以指定标签选择器以进一步过滤该卷集。 只有标签与选择器匹配的卷才能绑定到声明。 选择器可以由两个字段组成：

matchLabels - 卷必须具有带此值的标签
matchExpressions - 通过指定关键字和值的关键字，值列表和运算符所做的要求列表。 有效运算符包括In，NotIn，Exists和DoesNotExist。
1
2
所有来自matchLabels和matchExpressions的要求都与AND一起使用，所有这些要求都必须满足才能匹配。

Class
声明可以通过使用属性storageClassName指定StorageClass的名称来请求特定的类。只有所请求的类的PV，与PVC相同的storageClassName的PV可以绑定到PVC。

PVC不一定要求一个className。它的storageClassName设置为等于“”的PVC总是被解释为请求没有类的PV，因此它只能绑定到没有类的PV（没有注释或一个等于“”）。没有storageClassName的PVC不完全相同，并且根据是否启用了DefaultStorageClass入门插件，集群的处理方式不同。

如果接纳插件已打开，则管理员可以指定默认的StorageClass。没有storageClassName的所有PVC只能绑定到该默认的PV。通过将StorageClass对象中的annotation storageclass.kubernetes.io/is-default-class设置为“true”来指定默认的StorageClass。如果管理员没有指定默认值，则集群会对PVC创建做出响应，就像入门插件被关闭一样。如果指定了多个默认值，则验收插件禁止创建所有PVC。 
如果入门插件已关闭，则不存在默认StorageClass的概念。没有storageClassName的所有PVC只能绑定到没有类的PV。在这种情况下，没有storageClassName的PVC的处理方式与将其storageClassName设置为“”的PVC相同。

根据安装方法，安装过程中可以通过addon manager在Kubernetes群集中部署默认的StorageClass。

当PVC指定一个选择器，除了请求一个StorageClass之外，这些要求被AND组合在一起：只有所请求的类和所请求的标签的PV可能被绑定到PVC。请注意，当前，具有非空选择器的PVC不能为其动态配置PV。

在过去，使用了注释volume.beta.kubernetes.io/storage-class，而不是storageClassName属性。该注释仍然可以工作，但在未来的Kubernetes版本中它将不被支持。


声明PVC作为Volumes
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