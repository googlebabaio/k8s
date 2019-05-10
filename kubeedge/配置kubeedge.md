<!-- toc -->
# 配置kubeedge
![](assets/markdown-img-paste-20190424233749805.png)

kubeedge分为云端(cloud)和边缘端(edge)

## 获取代码

```
git clone https://github.com/kubeedge/kubeedge.git $GOPATH/src/github.com/kubeedge/kubeedge
cd $GOPATH/src/github.com/kubeedge/kubeedge
```

## 代码层级
```
[root@k8s-master kubeedge]# tree -L 1
.
├── build
├── cloud
├── common
├── CONTRIBUTING.md
├── docs
├── edge
├── external-dependency.md
├── Gopkg.lock
├── Gopkg.toml
├── LICENSE
├── MAINTAINERS
├── Makefile
├── README.md
├── README_zh.md
├── tests
└── vendor
```

主要看3个：`cloud`、`edge`、`build`



## 编译二进制文件

### 编译cloud和edge二进制文件
```
cd $GOPATH/src/github.com/kubeedge/kubeedge
make
```

### 编译cloud二进制文件
```
cd $GOPATH/src/github.com/kubeedge/kubeedge
make all WHAT=cloud
```
等同于:
```
cd $GOPATH/src/github.com/kubeedge/kubeedge/cloud/edgecontroller
make
```

### 编译edge二进制文件
```
cd $GOPATH/src/github.com/kubeedge/kubeedge
make all WHAT=edge
```
等同于:
```
cd $GOPATH/src/github.com/kubeedge/kubeedge/edge
make
```




## 部署云端(cloud)
云端是k8s的扩展，使用了CRD对k8s进行扩展，本文将以二进制的方式进行云端的部署

因为cloud是k8s的扩展,所以需要与k8s的master进行通信，我们一般都做了双向tls，所以为了简便，可以先暴露非安全的IP和端口用来测试
```
vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Add the following flags in spec: containers: -command section
- --insecure-port=8080
- --insecure-bind-address=0.0.0.0
```

当然，生产中，我们不建议这么用，我们还是老老实实的在control.yaml中配置kubeconfig的路径即可.

进入
```
[root@k8s-master cloud]# tree -L 3
.
├── edgecontroller
│   ├── cmd
│   │   └── edgecontroller.go
│   ├── conf
│   │   ├── controller.yaml
│   │   ├── edgecontroller.tar
│   │   ├── logging.yaml
│   │   └── modules.yaml
│   ├── edgecontroller
│   ├── edgecontroller.bak
│   ├── edgecontroller.log
│   ├── Makefile
│   └── pkg
│       ├── cloudhub
│       └── controller
└── README.md
```

进入到`$GOPATH/src/github.com/kubeedge/kubeedge/cloud/edgecontroller/conf`
修改`controller.yaml`的相关内容
```
[root@k8s-master conf]# cat controller.yaml
controller:
  kube:
    master: https://192.168.3.6:443 #配置了这个就不需要再使用非安全端口和非安全ip了
    namespace: ""
    content_type: "application/vnd.kubernetes.protobuf"
    qps: 5
    burst: 10
    node_update_frequency: 10
    kubeconfig: "/root/.kube/config"   #这个地方就写kubeconfig的地址
cloudhub:
  address: 0.0.0.0
  port: 10000
  ca: /sure/cert/rootCA.crt  # 可以通过脚本生成,只需要注意路径即可
  cert: /sure/cert/edge.crt  # 可以通过脚本生成,只需要注意路径即可
  key: /sure/cert/edge.key   # 可以通过脚本生成,只需要注意路径即可
  keepalive-interval: 20
  write-timeout: 20
  node-limit: 10
```


## 部署边缘端(edge)

此方式将部署 cloud 端到 k8s 集群，所以需要登录到 k8s 的 master 节点上（或者其他可以用 `kubectl` 操作集群的机器）。

存放在 `github.com/kubeedge/kubeedge/build/cloud` 里的各个编排文件和脚本会被用到。所以需要先将这些文件放到可以用 kubectl 操作的地方。

首先， 确保 k8s 集群可以拉到 edge controller 镜像。如果没有， 可以构建一个，然后推到集群能拉到的 registry 上。

```
cd $GOPATH/src/github.com/kubeedge/kubeedge
make cloudimage
```
> 这一步如果已经找到edgecontroller的镜像，就可以不用做了

然后，需要生成 tls 证书。这步成功的话，会生成 `06-secret.yaml`。

```
cd build/cloud
../tools/certgen.sh buildSecret | tee ./06-secret.yaml
```

接着，按照编排文件的文件名顺序创建各个 k8s 资源。在创建之前，应该检查每个编排文件内容，以确保符合特定的集群环境。

```
for resource in $(ls *.yaml); do kubectl create -f $resource; done
```

最后，基于`08-service.yaml.example`，创建一个适用于集群环境的 service，
将 cloud hub 暴露到集群外，让 edge core 能够连到。
