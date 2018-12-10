
<!-- toc --> 
# 部署master节点
## 1.准备安装包
```
[root@master ~]#  tar zxf kubernetes.tar.gz 
[root@master ~]# tar zxf kubernetes-server-linux-amd64.tar.gz 
[root@master ~]# tar zxf kubernetes-client-linux-amd64.tar.gz
[root@master ~]# tar zxf kubernetes-node-linux-amd64.tar.gz
[root@master ~]# cd /usr/local/src/kubernetes/server/bin
[root@master bin]# ls -l
total 1862792
-rwxr-xr-x 1 root root  59470358 Sep 10 02:44 apiextensions-apiserver
-rwxr-xr-x 1 root root 138267349 Sep 10 02:44 cloud-controller-manager
-rw-r--r-- 1 root root         8 Sep 10 02:44 cloud-controller-manager.docker_tag
-rw-r--r-- 1 root root 139661312 Sep 10 02:44 cloud-controller-manager.tar
-rwxr-xr-x 1 root root 227878608 Sep 10 02:44 hyperkube
-rwxr-xr-x 1 root root  57395147 Sep 10 02:44 kubeadm
-rwxr-xr-x 1 root root  58078146 Sep 10 02:44 kube-aggregator
-rw-r--r-- 1 root root         8 Sep 10 02:44 kube-aggregator.docker_tag
-rw-r--r-- 1 root root  59471872 Sep 10 02:44 kube-aggregator.tar
-rwxr-xr-x 1 root root 185513792 Sep 10 02:44 kube-apiserver
-rw-r--r-- 1 root root         8 Sep 10 02:44 kube-apiserver.docker_tag
-rw-r--r-- 1 root root 186907648 Sep 10 02:44 kube-apiserver.tar
-rwxr-xr-x 1 root root 154113026 Sep 10 02:44 kube-controller-manager
-rw-r--r-- 1 root root         8 Sep 10 02:44 kube-controller-manager.docker_tag
-rw-r--r-- 1 root root 155507200 Sep 10 02:44 kube-controller-manager.tar
-rwxr-xr-x 1 root root  55414436 Sep 10 02:44 kubectl
-rwxr-xr-x 1 root root 163041864 Sep 10 02:44 kubelet
-rwxr-xr-x 1 root root  52068695 Sep 10 02:44 kube-proxy
-rw-r--r-- 1 root root         8 Sep 10 02:44 kube-proxy.docker_tag
-rw-r--r-- 1 root root  99644928 Sep 10 02:44 kube-proxy.tar
-rwxr-xr-x 1 root root  55636875 Sep 10 02:44 kube-scheduler
-rw-r--r-- 1 root root         8 Sep 10 02:44 kube-scheduler.docker_tag
-rw-r--r-- 1 root root  57030656 Sep 10 02:44 kube-scheduler.tar
-rwxr-xr-x 1 root root   2330265 Sep 10 02:44 mounter
[root@master bin]# cp kube-apiserver kube-controller-manager kube-scheduler /etc/kubernetes/bin
```

## 2.创建生成kubernetes的CSR的 JSON 配置文件
内容和etcd-csr.json非常类似
```
[root@master temp]# cat kubernetes-csr.json 
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "172.18.53.221",
    "10.1.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

## 3.生成 kubernetes 证书和私钥
```
[root@master temp]# cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
    -ca-key=/etc/kubernetes/ssl/ca-key.pem \
    -config=/etc/kubernetes/ssl/ca-config.json \
    -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
2018/09/20 13:51:56 [INFO] generate received request
2018/09/20 13:51:56 [INFO] received CSR
2018/09/20 13:51:56 [INFO] generating key: rsa-2048
2018/09/20 13:51:56 [INFO] encoded CSR
2018/09/20 13:51:56 [INFO] signed certificate with serial number 459304795069308189390350956282652310628094139983
2018/09/20 13:51:56 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
[root@master temp]# 
[root@master temp]# ls -l kube*
-rw-r--r-- 1 root root 1245 Sep 20 13:51 kubernetes.csr
-rw-r--r-- 1 root root  435 Sep 20 13:49 kubernetes-csr.json
-rw------- 1 root root 1679 Sep 20 13:51 kubernetes-key.pem
-rw-r--r-- 1 root root 1610 Sep 20 13:51 kubernetes.pem
```

## 4.分发证书
```
[root@master temp]# cp kubernetes*.pem ..
[root@master temp]# scp kubernetes*.pem node1:/etc/kubernetes/ssl  
[root@master temp]# scp kubernetes*.pem node2:/etc/kubernetes/ssl
[root@master temp]# 
```

## 5.配置 kube-apiserver 
### 5.1 创建 kube-apiserver 使用的客户端 token 文件
```
[root@master temp]# head -c 16 /dev/urandom | od -An -t x | tr -d ' '
cdf6d36d697dae799728ac52531a43a8
[root@master temp]# vim /etc/kubernetes/ssl/bootstrap-token.csv
[root@master temp]# cat /etc/kubernetes/ssl/bootstrap-token.csv
cdf6d36d697dae799728ac52531a43a8,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
[root@master temp]# 
```

### 5.2 创建基础用户名/密码认证配置(abac)
```
[root@master temp]# vim /etc/kubernetes/ssl/basic-auth.csv
[root@master temp]# cat /etc/kubernetes/ssl/basic-auth.csv
admin123,admin,1
readonly123,readonly,2
```
注意格式是csv的，顺序是：[密码，用户名，序号]

### 5.3 部署kube-apiserver服务
```
[root@master temp]# cat /usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/etc/kubernetes/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \
  --bind-address=172.18.53.221 \
  --insecure-bind-address=127.0.0.1 \
  --authorization-mode=Node,RBAC \
  --runtime-config=rbac.authorization.k8s.io/v1 \
  --kubelet-https=true \
  --anonymous-auth=false \
  --basic-auth-file=/etc/kubernetes/ssl/basic-auth.csv \
  --enable-bootstrap-token-auth \
  --token-auth-file=/etc/kubernetes/ssl/bootstrap-token.csv \
  --service-cluster-ip-range=10.1.0.0/16 \
  --service-node-port-range=20000-40000 \
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \
  --etcd-servers=https://172.18.53.221:2379,https://172.18.53.223:2379,https://172.18.53.224:2379 \
  --enable-swagger-ui=true \
  --allow-privileged=true \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/etc/kubernetes/log/api-audit.log \
  --event-ttl=1h \
  --v=2 \
  --logtostderr=false \
  --log-dir=/etc/kubernetes/log
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

> 注意：目前 k8s 支持授权插件：
* ABAC
* RBAC
* Webhook
* Node

```
仅v1.7版本以上支持Node授权，配合NodeRestriction准入控制来限制kubelet仅可访问node、endpoint、pod、service以及secret、configmap、PV和PVC等相关的资源，配置方法为
--authorization-mode=Node,RBAC --admission-control=...,NodeRestriction,...
注意，kubelet认证需要使用system:nodes组，并使用用户名system:node:<nodeName>。
```

### 5.4 启动kube-apiserver服务
```
[root@master temp]# systemctl daemon-reload
[root@master temp]# systemctl enable kube-apiserver
[root@master temp]# systemctl start kube-apiserver
```

### 5.5 查看kube-apiserver服务状态
```
[root@master temp]# systemctl status kube-apiserver
● kube-apiserver.service - Kubernetes API Server
   Loaded: loaded (/usr/lib/systemd/system/kube-apiserver.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2018-09-20 14:05:52 CST; 7s ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 1601 (kube-apiserver)
   CGroup: /system.slice/kube-apiserver.service
           └─1601 /etc/kubernetes/bin/kube-apiserver --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction --bind-address=17...

Sep 20 14:05:43 master systemd[1]: Starting Kubernetes API Server...
Sep 20 14:05:43 master kube-apiserver[1601]: Flag --admission-control has been deprecated, Use --enable-admission-plugins or --disable-admission-plugins instead. Will be remo...ure version.
Sep 20 14:05:43 master kube-apiserver[1601]: Flag --insecure-bind-address has been deprecated, This flag will be removed in a future version.
Sep 20 14:05:47 master kube-apiserver[1601]: [restful] 2018/09/20 14:05:47 log.go:33: [restful/swagger] listing is available at https://172.18.53.221:6443/swaggerapi
Sep 20 14:05:47 master kube-apiserver[1601]: [restful] 2018/09/20 14:05:47 log.go:33: [restful/swagger] https://172.18.53.221:6443/swaggerui/ is mapped to folder /swagger-ui/
Sep 20 14:05:49 master kube-apiserver[1601]: [restful] 2018/09/20 14:05:49 log.go:33: [restful/swagger] listing is available at https://172.18.53.221:6443/swaggerapi
Sep 20 14:05:49 master kube-apiserver[1601]: [restful] 2018/09/20 14:05:49 log.go:33: [restful/swagger] https://172.18.53.221:6443/swaggerui/ is mapped to folder /swagger-ui/
Sep 20 14:05:52 master systemd[1]: Started Kubernetes API Server.
Hint: Some lines were ellipsized, use -l to show in full.
```
## 6.配置kube-controller-manager
### 6.1 部署kube-controller-manager服务
```
[root@master temp]# cat /usr/lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/etc/kubernetes/bin/kube-controller-manager \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --allocate-node-cidrs=true \
  --service-cluster-ip-range=10.1.0.0/16 \
  --cluster-cidr=10.2.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --leader-elect=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/etc/kubernetes/log

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 6.2 启动kube-controller-manager服务
```
[root@master temp]# systemctl daemon-reload
[root@master temp]# systemctl enable kube-controller-manager
[root@master temp]# systemctl start kube-controller-manager
```

### 6.3 查看kube-controller-manager服务状态
```
[root@master temp]# systemctl status kube-controller-manager
● kube-controller-manager.service - Kubernetes Controller Manager
   Loaded: loaded (/usr/lib/systemd/system/kube-controller-manager.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2018-09-20 14:09:54 CST; 6s ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 1654 (kube-controller)
   CGroup: /system.slice/kube-controller-manager.service
           └─1654 /etc/kubernetes/bin/kube-controller-manager --address=127.0.0.1 --master=http://127.0.0.1:8080 --allocate-node-cidrs=true --service-cluster-ip-range=10.1.0.0/16 --clust...

Sep 20 14:09:54 master systemd[1]: Started Kubernetes Controller Manager.
Sep 20 14:09:54 master systemd[1]: Starting Kubernetes Controller Manager...
```
## 7.配置kube-scheduler
### 7.1 部署kube-scheduler服务
```
[root@master temp]# cat /usr/lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/etc/kubernetes/bin/kube-scheduler \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --leader-elect=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/etc/kubernetes/log

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target	
```

### 7.2 启动 kube-scheduler 服务
```
[root@master temp]#  systemctl daemon-reload
[root@master temp]# systemctl enable kube-scheduler
[root@master temp]# systemctl start kube-scheduler
```

### 7.3 查看 kube-scheduler 状态
```
[root@master temp]# systemctl status kube-scheduler
● kube-scheduler.service - Kubernetes Scheduler
   Loaded: loaded (/usr/lib/systemd/system/kube-scheduler.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2018-09-20 14:12:01 CST; 4s ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 1711 (kube-scheduler)
   CGroup: /system.slice/kube-scheduler.service
           └─1711 /etc/kubernetes/bin/kube-scheduler --address=127.0.0.1 --master=http://127.0.0.1:8080 --leader-elect=true --v=2 --logtostderr=false --log-dir=/etc/kubernetes/log

Sep 20 14:12:01 master systemd[1]: Started Kubernetes Scheduler.
Sep 20 14:12:01 master systemd[1]: Starting Kubernetes Scheduler...
```

