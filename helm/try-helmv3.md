<!-- toc -->

记录一下helm v3版本的新特性以及试用情况
helm v2与v3可以共存,大家都用的是`~/.helm`
<!--more-->



## helm v3的特性
最大特性就是去掉了tiller，使用``~/.kube/config`作为与集群的通信的桥梁


在 Helm 2 中，Tiller 是作为一个 Deployment 部署在 kube-system 命名空间中，很多情况下，我们会为 Tiller 准备一个 ServiceAccount ，这个 ServiceAccount 通常拥有集群的所有权限。用户可以使用本地 Helm 命令，自由地连接到 Tiller 中并通过 Tiller 创建、修改、删除任意命名空间下的任意资源。

然而在多租户场景下，这种方式也会带来一些安全风险，我们即要对这个 ServiceAccount 做很多剪裁，又要单独控制每个租户的控制，这在当前的 Tiller 模式下看起来有些不太可能。

于是在 Helm 3 中，Tiller 被移除了。新的 Helm 客户端会像 kubectl 命令一样，读取本地的 kubeconfig 文件，使用我们在 kubeconfig 中预先定义好的权限来进行一系列操作。这样做法即简单，又安全。

虽然 Tiller 文件被移除了，但 Release 的信息仍在集群中以 ConfigMap 的方式存储，因此体验和 Helm 2 没有区别。


## V3带来的变化
### Release 不再是全局资源
在 Helm 2 中，Tiller 自身部署往往在 kube-system 下，虽然不一定是 cluster-admin 的全局管理员权限，但是一般都会有 kube-system 下的权限。
当 Tiller 想要存储一些信息的时候，它被设计成在 kube-system 下读写 ConfigMap 。

在 Helm 3 中，Helm 客户端使用 kubeconfig 作为认证信息直接连接到 Kubernetes APIServer，不一定拥有 cluster-admin 权限或者写 kube-system 的权限，因此它只能将需要存储的信息存在当前所操作的 Kubernetes Namespace 中，继而 Release 变成了一种命名空间内的资源。

Helm v2存储的release 都在tiller 所在的namespace,而且是以cm进行存储的。
Helm v3，所有的信息都存储在release对应的namespace 下，而且以secret存储。这是v2和v3很不相同的地方。

### Values 支持 `JSON Schema` 校验器
Helm Charts 是一堆 Go Template 文件、一个变量文件 Values 和一些 Charts 描述文件的组合。Go Template 和 Kubernetes 资源描述文件的内容十分灵活，在开发迭代过程中，很容易出现一些变量未定义的问题。Helm 3 引入了 JSON Schema 校验，它支持用一长串 DSL 来描述一个变量文件的格式、检查所有输入的变量的格式。

当我们运行 `helm install` 、 `helm upgrade` 、 `helm lint` 、 `helm template` 命令时，JSON Schema 的校验会自动运行，如果失败就会立即报错。

### 推送 Charts 到容器镜像仓库中
互联网上有一些 Helm Charts 托管平台，负责分发常见的开源应用，例如 `Helm Hub`、`KubeApps Hub`。

在企业环境中，用户需要私有化的 Helm Charts 托管。最常见的方案是部署 ChartMuseum，使其对接一些云存储。当然也有一些较为小众的方案，比如 [App-Regsitry]() 直接复用了 Docker 镜像仓库 Registry V2，再比如 helm-s3 直接通过云存储 S3 存取 Charts 文件。

如今在 Helm 3，Helm 直接支持了推送 Charts 到容器镜像仓库的能力，希望支持满足 OCI 标准的所有 Registry。但这部分还没有完全被开发完，会在未来的 alpha 版本中被完善。

```
docker run -dp 5000:5000 --restart=always --name registry registry:2

然后下载一个chart包helm fetch stable/wordpress

# helmv3 chart save wordpress localhost:5000/wordpress:latest
Name: wordpress
Version: 0.6.13
Meta: sha256:83c48dd3c01a2952066ead67023ea14963a88db4287650baad5ea1ddd8ff9590
Content: sha256:248c8c68f4f614003c8b1a9d78787e5f07e979e9b996981df993cf380f498c97
latest: saved


# ./helmv3 chart list
REF                                NAME         VERSION    DIGEST     SIZE        CREATED
localhost:5000/wordpress:latest    wordpress    0.6.13     248c8c6    12.0 KiB    11 seconds


# helmv3 chart push localhost:5000/wordpress:latest
The push refers to repository [localhost:5000/wordpress]
Name: wordpress
Version: 0.6.13
Meta: sha256:83c48dd3c01a2952066ead67023ea14963a88db4287650baad5ea1ddd8ff9590
Content: sha256:248c8c68f4f614003c8b1a9d78787e5f07e979e9b996981df993cf380f498c97
latest: pushed to remote (2 layers, 12.6 KiB total)
```

### 代码复用 - Library Chart 支持
Helm 3 中引入了一种新的 Chart 类型，名为 Library Chart 。它不会部署出一些具体的资源，只能被其他的 Chart 所引用，提高代码的可用复用性。当一个 Chart 想要使用该 Library Chart 内的一些模板时，可以在 Chart.yaml 的 dependencies 依赖项中指定。

### 其他的一些变化

- Go Import 的路径由 k8s.io/helm 变成了 helm.sh/helm
- 简化了 Chart 内置变量 Capabilities 的一些属性
- helm install 不再默认生成一个 Release 的名称，除非指定了 --generate-name 
- 移除了用于本地临时搭建 Chart Repository 的 helm serve 命令
- helm delete 更名为 helm uninstall ， helm inspect 更名为 helm show ， helm fetch 更名为 helm pull ，但以上旧的命令当前仍能使用
- requirements.yaml 被整合到了 Chart.yaml 中，但格式保持不变



## 安装过程
安装过程变得非常简单了,只需要安装client即可
```
# wget https://cloudnativeapphub.oss-cn-hangzhou.aliyuncs.com/helm-v3.0.0-alpha.1-linux-amd64.tar.gz
--2019-07-10 09:50:39--  https://cloudnativeapphub.oss-cn-hangzhou.aliyuncs.com/helm-v3.0.0-alpha.1-linux-amd64.tar.gz
正在解析主机 cloudnativeapphub.oss-cn-hangzhou.aliyuncs.com (cloudnativeapphub.oss-cn-hangzhou.aliyuncs.com)... 118.31.219.223
正在连接 cloudnativeapphub.oss-cn-hangzhou.aliyuncs.com (cloudnativeapphub.oss-cn-hangzhou.aliyuncs.com)|118.31.219.223|:443... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度：12711766 (12M) [application/x-gzip]
正在保存至: “helm-v3.0.0-alpha.1-linux-amd64.tar.gz”

100%[=======================================================================================================================>] 12,711,766  1.31MB/s 用时 9.1s

2019-07-10 09:50:53 (1.34 MB/s) - 已保存 “helm-v3.0.0-alpha.1-linux-amd64.tar.gz” [12711766/12711766])

# ls
helm-v3.0.0-alpha.1-linux-amd64.tar.gz
# tar -zxf helm-v3.0.0-alpha.1-linux-amd64.tar.gz
# cd linux-amd64/
# mv helm /usr/local/bin/helmv3
```

## 使用过程
### 添加仓库
与V2是共享的!
```
# helm repo list
NAME            URL
local           http://127.0.0.1:8879/charts
microsoft       https://mirror.azure.cn/kubernetes/charts/

# helmv3 repo add ali-apphub https://apphub.aliyuncs.com
"ali-apphub" has been added to your repositories

# helm repo list
NAME            URL
local           http://127.0.0.1:8879/charts
microsoft       https://mirror.azure.cn/kubernetes/charts/
ali-apphub      https://apphub.aliyuncs.com
```

### 安装应用

`helmV2`安装的应用`helmV3`看不见，反之亦然！

#### 先使用v2安装一次
```
# helm install ali-apphub/guestbook --set service.type=NodePort
NAME:   invisible-worm
LAST DEPLOYED: Wed Jul 10 09:57:57 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME                      READY  UP-TO-DATE  AVAILABLE  AGE
invisible-worm-guestbook  0/2    0           0          0s
redis-master              0/1    0           0          0s
redis-slave               0/2    0           0          0s

==> v1/Service
NAME                      TYPE          CLUSTER-IP      EXTERNAL-IP  PORT(S)         AGE
invisible-worm-guestbook  LoadBalancer  10.101.48.210   <pending>    3000:25883/TCP  0s
redis-master              ClusterIP     10.105.253.249  <none>       6379/TCP        0s
redis-slave               ClusterIP     10.105.98.176   <none>       6379/TCP        0s


NOTES:
1. Get the application URL by running these commands:
  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        You can watch the status of by running 'kubectl get svc -w invisible-worm-guestbook --namespace default'
  export SERVICE_IP=$(kubectl get svc --namespace default invisible-worm-guestbook -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo http://$SERVICE_IP:3000
```

安装完成后查看信息
```
# helm status invisible-worm
LAST DEPLOYED: Wed Jul 10 09:57:57 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME                      READY  UP-TO-DATE  AVAILABLE  AGE
invisible-worm-guestbook  2/2    2           2          13m
redis-master              1/1    1           1          13m
redis-slave               2/2    2           2          13m

==> v1/Pod(related)
NAME                                     READY  STATUS   RESTARTS  AGE
invisible-worm-guestbook-cbc8b5c8-bwwg4  2/2    Running  0         13m
invisible-worm-guestbook-cbc8b5c8-xb492  2/2    Running  0         13m
redis-master-9954bc48d-w4kvk             2/2    Running  0         13m
redis-slave-75f9c8bc49-msgnm             2/2    Running  0         13m
redis-slave-75f9c8bc49-wzn5q             2/2    Running  0         13m

==> v1/Service
NAME                      TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)         AGE
invisible-worm-guestbook  NodePort   10.101.48.210   <none>       3000:25883/TCP  13m
redis-master              ClusterIP  10.105.253.249  <none>       6379/TCP        13m
redis-slave               ClusterIP  10.105.98.176   <none>       6379/TCP        13m

NOTES:
1. Get the application URL by running these commands:
  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        You can watch the status of by running 'kubectl get svc -w invisible-worm-guestbook --namespace default'
  export SERVICE_IP=$(kubectl get svc --namespace default invisible-worm-guestbook -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo http://$SERVICE_IP:3000

```

而用V3是看不见这个已经安装的应用的
```
# helmv3 list
NAME    NAMESPACE       REVISION        UPDATED STATUS  CHART
```

#### 再使用v3安装一次
```
# helmv3 install ali-apphub/guestbook --set service.type=NodePort
Error: must either provide a name or specify --generate-name
```

V3的一个特性就是必需要给应用娶一个名字，或者使用`--generate-name`选项来自动生成一个名字.

加上名字,重新安装
```
#  helmv3 install gusetbook  ali-apphub/guestbook --set service.type=NodePort
NAME: gusetbook
LAST DEPLOYED: 2019-07-10 10:13:20.098413164 +0800 CST m=+5.744019735
NAMESPACE: sure
STATUS: deployed

NOTES:
1. Get the application URL by running these commands:
  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        You can watch the status of by running 'kubectl get svc -w gusetbook-guestbook --namespace sure'
  export SERVICE_IP=$(kubectl get svc --namespace sure gusetbook-guestbook -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo http://$SERVICE_IP:3000
```

安装完成后查看信息:
```
# helmv3 status  gusetbook
NAME: gusetbook
LAST DEPLOYED: 2019-07-10 10:13:20.098413164 +0800 CST
NAMESPACE: sure
STATUS: deployed

NOTES:
1. Get the application URL by running these commands:
  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        You can watch the status of by running 'kubectl get svc -w gusetbook-guestbook --namespace sure'
  export SERVICE_IP=$(kubectl get svc --namespace sure gusetbook-guestbook -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo http://$SERVICE_IP:3000
```

## 其他一些命令
- `helm delete` 更名为 `helm uninstall` 
- `helm inspect` 更名为 `helm show` 
- `helm fetch` 更名为 `helm pull`
- 但以上旧的命令当前仍能使用

## roadmap
- alpha1: 移除Tiller， 提供Library charts, 存储格式改成secret， 开始OCI集成工作
- alpha2: 更好的 OCI 集成，Lua 模板支持
- alpha3: 重构更新策略（可能是客户端侧进行，也可能是服务端侧进行）
