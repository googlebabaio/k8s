<!-- toc --> 
## 一、准备工作
### 1. 准备node节点的kubelet和kube-proxy二进制包
```
[root@master bin]# pwd
/usr/local/src/kubernetes/server/bin
[root@master bin]# ll
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
[root@master bin]# scp kubelet kube-proxy node1:/etc/kubernetes/bin  
[root@master bin]# scp kubelet kube-proxy node2:/etc/kubernetes/bin  
[root@master bin]# 
```

### 2.创建角色绑定
```
[root@master bin]# kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
clusterrolebinding.rbac.authorization.k8s.io/kubelet-bootstrap created
```

### 3.创建 用于kubelet的kubeconfig 文件 
#### 3.1 设置集群参数
```
 kubectl config set-cluster kubernetes \
   --certificate-authority=/etc/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://172.18.53.221:6443 \
   --kubeconfig=bootstrap.kubeconfig
```

#### 3.2 设置客户端认证参数
```
kubectl config set-credentials kubelet-bootstrap \
  --token=cdf6d36d697dae799728ac52531a43a8 \
  --kubeconfig=bootstrap.kubeconfig   
```

#### 3.3 设置上下文参数
```
kubectl config set-context default \
   --cluster=kubernetes \
   --user=kubelet-bootstrap \
   --kubeconfig=bootstrap.kubeconfig
```


#### 3.4 选择默认上下文
```
 kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```

#### 3.5 将当前目录生成的 `bootstrap.kubeconfig` 分发到node节点上
```
[root@master bin]# cp bootstrap.kubeconfig /etc/kubernetes/cfg
[root@master bin]# scp /etc/kubernetes/cfg/bootstrap.kubeconfig node1:/etc/kubernetes/cfg 
[root@master bin]# scp /etc/kubernetes/cfg/bootstrap.kubeconfig node2:/etc/kubernetes/cfg
```

## 二、部署kubelet
### 1.设置CNI支持
```
[root@node1 ~]# mkdir -p /etc/cni/net.d
[root@node1 ~]# vim /etc/cni/net.d/10-default.conf
[root@node1 ~]# cat /etc/cni/net.d/10-default.conf
{
        "name": "flannel",
        "type": "flannel",
        "delegate": {
            "bridge": "docker0",
            "isDefaultGateway": true,
            "mtu": 1400
        }
}
```

### 2.创建kubelet目录
```
[root@node1 ~]# mkdir /var/lib/kubelet
```
### 3.创建kubelet服务
创建kubelet服务配置
```
[root@node1 ~]# vim /usr/lib/systemd/system/kubelet.service
[root@node1 ~]# cat  /usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/etc/kubernetes/bin/kubelet \
  --address=172.18.53.223 \
  --hostname-override=172.18.53.223 \
  --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.0 \
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/cfg/bootstrap.kubeconfig \
  --kubeconfig=/etc/kubernetes/cfg/kubelet.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --network-plugin=cni \
  --cni-conf-dir=/etc/cni/net.d \
  --cni-bin-dir=/etc/kubernetes/bin/cni \
  --cluster-dns=10.1.0.2 \
  --cluster-domain=cluster.local. \
  --hairpin-mode=hairpin-veth \
  --allow-privileged=true \
  --fail-swap-on=false \
  --v=2 \
  --logtostderr=false \
  --log-dir=/etc/kubernetes/log
Restart=on-failure
RestartSec=5
```

### 4.启动kubelet服务
```
[root@node1 ~]# systemctl daemon-reload
[root@node1 ~]# systemctl enable kubelet
[root@node1 ~]# systemctl start kubelet
```

### 5.查看kubelet服务状态
```
[root@node1 ~]# systemctl status kubelet
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; static; vendor preset: disabled)
   Active: active (running) since Thu 2018-09-20 15:54:03 CST; 4s ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 24798 (kubelet)
   Memory: 14.2M
   CGroup: /system.slice/kubelet.service
           └─24798 /etc/kubernetes/bin/kubelet --address=172.18.53.223 --hostname-override=172.18.53.223 --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.0 --experimental...

Sep 20 15:54:03 node1 systemd[1]: Started Kubernetes Kubelet.
Sep 20 15:54:03 node1 systemd[1]: Starting Kubernetes Kubelet...
Sep 20 15:54:03 node1 kubelet[24798]: Flag --address has been deprecated, This parameter should be set via the config file specified by the Kubelet's --config flag. See http... information.
Sep 20 15:54:03 node1 kubelet[24798]: Flag --experimental-bootstrap-kubeconfig has been deprecated, Use --bootstrap-kubeconfig
Sep 20 15:54:03 node1 kubelet[24798]: Flag --cluster-dns has been deprecated, This parameter should be set via the config file specified by the Kubelet's --config flag. See ... information.
Sep 20 15:54:03 node1 kubelet[24798]: Flag --cluster-domain has been deprecated, This parameter should be set via the config file specified by the Kubelet's --config flag. S... information.
Sep 20 15:54:03 node1 kubelet[24798]: Flag --hairpin-mode has been deprecated, This parameter should be set via the config file specified by the Kubelet's --config flag. See... information.
Sep 20 15:54:03 node1 kubelet[24798]: Flag --allow-privileged has been deprecated, will be removed in a future version
Sep 20 15:54:03 node1 kubelet[24798]: Flag --fail-swap-on has been deprecated, This parameter should be set via the config file specified by the Kubelet's --config flag. See... information.
Hint: Some lines were ellipsized, use -l to show in full.
```

以上步骤从 `设置CNI支持`开始到 `查看kubelet服务状态`，需要在node2节点也做一次

### 6.在master节点查看csr请求
```
[root@master bin]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-2LtbtsOORfUJd6ZGNZyVjmbHO0eyu619JxbRyfXySuA   14s       kubelet-bootstrap   Pending
node-csr-7YxIy4ylo9KNGWDzs-oduDGfQP9kULlaXaL8B1E4X34   16s       kubelet-bootstrap   Pending
```

### 7.在master节点通过kubelet 的 TLS 证书请求
```
[root@master bin]# kubectl get csr|grep 'Pending' | awk 'NR>0{print $1}'| xargs kubectl certificate approve
certificatesigningrequest.certificates.k8s.io/node-csr-2LtbtsOORfUJd6ZGNZyVjmbHO0eyu619JxbRyfXySuA approved
certificatesigningrequest.certificates.k8s.io/node-csr-7YxIy4ylo9KNGWDzs-oduDGfQP9kULlaXaL8B1E4X34 approved
```
再次查询状态已经是approved的了
```
[root@master bin]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-2LtbtsOORfUJd6ZGNZyVjmbHO0eyu619JxbRyfXySuA   58s       kubelet-bootstrap   Approved,Issued
node-csr-7YxIy4ylo9KNGWDzs-oduDGfQP9kULlaXaL8B1E4X34   1m        kubelet-bootstrap   Approved,Issued
```
这个时候查询`get node`也可以看到结果了
```
[root@master bin]# kubectl get node
NAME            STATUS    ROLES     AGE       VERSION
172.18.53.223   Ready     <none>    19s       v1.11.3
172.18.53.224   Ready     <none>    19s       v1.11.3
```

## 三、配置kube-proxy
### 1.安装kube-proxy使用LVS需要的包
注意：如果不使用lvs的话，就会使用iptables，这个是1.10版本的新特性
```
yum install -y ipvsadm ipset conntrack
```

### 2.创建 kube-proxy 证书请求的json
```
[root@master temp]# cat kube-proxy-csr.json 
{
  "CN": "system:kube-proxy",
  "hosts": [],
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

### 3.生成kube-proxy证书
```
[root@master temp]# cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
    -ca-key=/etc/kubernetes/ssl/ca-key.pem \
    -config=/etc/kubernetes/ssl/ca-config.json \
    -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
2018/09/20 16:08:15 [INFO] generate received request
2018/09/20 16:08:15 [INFO] received CSR
2018/09/20 16:08:15 [INFO] generating key: rsa-2048
2018/09/20 16:08:16 [INFO] encoded CSR
2018/09/20 16:08:16 [INFO] signed certificate with serial number 372779783324557452959035572248079920613862154571
2018/09/20 16:08:16 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
[root@master temp]# ls -l kube-pro*.pem
-rw------- 1 root root 1675 Sep 20 16:08 kube-proxy-key.pem
-rw-r--r-- 1 root root 1403 Sep 20 16:08 kube-proxy.pem
```

### 4.分发证书
```
[root@master temp]# scp kube-proxy*.pem node1:/etc/kubernetes/ssl/ 
[root@master temp]# scp kube-proxy*.pem node2:/etc/kubernetes/ssl/
```

### 5.创建kube-proxy配置文件
#### 5.1 创建集群信息
```
kubectl config set-cluster kubernetes \
   --certificate-authority=/etc/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://172.18.53.221:6443 \
   --kubeconfig=kube-proxy.kubeconfig
```

#### 5.2 配置认证信息
```
kubectl config set-credentials kube-proxy \
 --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
 --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
 --embed-certs=true \
 --kubeconfig=kube-proxy.kubeconfig
```

#### 5.3设置上下文环境
```
kubectl config set-context default \
   --cluster=kubernetes \
   --user=kube-proxy \
   --kubeconfig=kube-proxy.kubeconfig
```

#### 5.4配置默认的上下文
```
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

#### 5.5分发kube-proxy.kubeconfig到两个node节点
```
[root@master temp]# cp kube-proxy.kubeconfig /etc/kubernetes/cfg
[root@master temp]# scp /etc/kubernetes/cfg/kube-proxy.kubeconfig node1:/etc/kubernetes/cfg/kube-proxy.kubeconfig
[root@master temp]# scp /etc/kubernetes/cfg/kube-proxy.kubeconfig node2:/etc/kubernetes/cfg/kube-proxy.kubeconfig
```

### 6.创建kube-proxy的工作目录
```
mkdir /var/lib/kube-proxy
```

### 7.创建kube-proxy服务配置
```
[root@node1 ~]# cat /usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/etc/kubernetes/bin/kube-proxy \
  --bind-address=172.18.53.223 \
  --hostname-override=172.18.53.223 \
  --kubeconfig=/etc/kubernetes/cfg/kube-proxy.kubeconfig \
  --masquerade-all \
  --feature-gates=SupportIPVSProxyMode=true \
  --proxy-mode=ipvs \
  --ipvs-min-sync-period=5s \
  --ipvs-sync-period=5s \
  --ipvs-scheduler=rr \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/etc/kubernetes/log

Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### 8.启动kube-proxy服务
```
[root@node1 ~]# systemctl daemon-reload
[root@node1 ~]# systemctl enable kube-proxy
Created symlink from /etc/systemd/system/multi-user.target.wants/kube-proxy.service to /usr/lib/systemd/system/kube-proxy.service.
[root@node1 ~]# systemctl start kube-proxy
```

### 9.查看kube-proxy服务状态
```
[root@node1 ~]# systemctl status kube-proxy
● kube-proxy.service - Kubernetes Kube-Proxy Server
   Loaded: loaded (/usr/lib/systemd/system/kube-proxy.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2018-09-20 16:27:38 CST; 11s ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 25355 (kube-proxy)
   Memory: 8.0M
   CGroup: /system.slice/kube-proxy.service
           ‣ 25355 /etc/kubernetes/bin/kube-proxy --bind-address=172.18.53.223 --hostname-override=172.18.53.223 --kubeconfig=/etc/kubernetes/cfg/kube-proxy.kubeconfig --masquerade-all -...

Sep 20 16:27:38 node1 systemd[1]: Started Kubernetes Kube-Proxy Server.
Sep 20 16:27:38 node1 systemd[1]: Starting Kubernetes Kube-Proxy Server...
```

### 10.检查LVS状态
```
[root@node1 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.1.0.1:443 rr
  -> 172.18.53.221:6443           Masq    1      0          0  
```
