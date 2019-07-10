在一个多用户、多团队的k8s集群上，通常会遇到一个问题，如何在不同团队之间取得资源的公平，即，不会因为某个流氓团队占据了所有资源，从而导致其他团队无法使用k8s。

k8s的解决方法是，通过RBAC将不同团队（or 项目）限制在不同的namespace下，通过resourceQuota来限制该namespace能够使用的资源。资源分为以下三种。

- 计算资源配额：cpu，memory
- 存储资源配置：requests.storage（真～总量），pvc，某storage class下的限制（例如针对fast类型的sc的限制）
- 对象数量配置：cm，service，pod的个数。



官方文档请见下面的Ref:
https://kubernetes.io/docs/tasks/administer-cluster/quota-memory-cpu-namespace/#clean-up

下面举个例子:
首先，集群管理员需要为团队创建一个namespace
```
kubectl create namespace yfzx
```

然后，将下面的quota apply到该namespace。
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    limits.cpu: "16"
    limits.memory: 8Gi
    persistentvolumeclaims: "8"
    pods: "16"
    requests.cpu: "8"
    requests.memory: 4Gi
    requests.storage: 30Gi
```

创建pod时指定相应的requests
```
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-demo
spec:
  containers:
  - name: quota-mem-cpu-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "16Gi"
        cpu: "800m"
      requests:
        memory: "4Gi"
        cpu: "500m"
```

> 注意，当开启了resource quota时，用户创建pod，必须指定cpu、内存的 requests or limits ，否则创建失败。resourceQuota搭配 limitRanges口感更佳：limitRange可以配置创建Pod的默认limit/request。
