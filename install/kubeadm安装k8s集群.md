<!-- toc -->
# kubeadm安装k8s集群

每个节点都要安装kubelet kubeadm


## 主机准备工作

### 修改主机名
```
# cat /etc/hosts
127.0.0.1	localhost	localhost.localdomain	localhost4	localhost4.localdomain4
::1	localhost	localhost.localdomain	localhost6	localhost6.localdomain6
172.26.243.111  k8s-node1
172.26.243.110  k8s-master
```

### 关闭防火墙

如果各个主机启用了防火墙，需要开放Kubernetes各个组件所需要的端口，可以查看Installing kubeadm中的”Check required ports”一节。 这里简单起见在各节点禁用防火墙：
```
systemctl stop firewalld.service     #停止firewall
systemctl disable firewalld.service #禁止firewall开机启动
firewall-cmd --state                       #查看防火墙状态
```

### 禁用SELINUX
```
setenforce 0
vi /etc/selinux/config #SELINUX修改为disabled SELINUX=disabled
```

### 关闭 swap
```
swapoff -a
```


## 安装docker
### 获取repo源(这个地方选用阿里的)
```
cd /etc/yum.repos.d/
wget  https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

### 用yum进行安装
```
yum install docker-ce -y
```

### 安装完成后启动
```
systemctl enable docker && systemctl start docker
```


## 安装kubelet/kubeadm/kubectl
- kubeadm: the command to bootstrap the cluster.
- kubelet: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
- kubectl: the command line util to talk to your cluster.


获取k8s的镜像
```
cat  <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

安装kubelet/kubeadm/kubectl
```
# yum install -y kubelet kubeadm kubectl
Installed:
  kubeadm.x86_64 0:1.14.1-0                                     kubectl.x86_64 0:1.14.1-0                                     kubelet.x86_64 0:1.14.1-0

Dependency Installed:
  conntrack-tools.x86_64 0:1.4.4-4.el7              cri-tools.x86_64 0:1.12.0-0                     kubernetes-cni.x86_64 0:0.7.5-0       libnetfilter_cthelper.x86_64 0:1.0.0-9.el7
  libnetfilter_cttimeout.x86_64 0:1.0.0-6.el7       libnetfilter_queue.x86_64 0:1.0.2-2.el7_2       socat.x86_64 0:1.7.3.2-2.el7

```

启动kubelet
```
systemctl enable kubelet && systemctl start kubelet
```

## 查看kubeadm init所需要的镜像
```
[root@k8s-master ~]# kubeadm config images list
I0425 15:55:58.500052   23338 version.go:96] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://dl.k8s.io/releas
e/stable-1.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)I0425 15:55:58.500133   23338 version.go:97] falling back to the local client version: v1.14.1
k8s.gcr.io/kube-apiserver:v1.14.1
k8s.gcr.io/kube-controller-manager:v1.14.1
k8s.gcr.io/kube-scheduler:v1.14.1
k8s.gcr.io/kube-proxy:v1.14.1
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
```

## 初始化master节点
```
kubeadm init --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers --pod-network-cidr=10.1.0.0/16 --service-cidr=10.2.0.0/16 --apiserver-advertise-address=172.26.243.110
```
注意几个选项：


参考：
https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#what-s-next

执行完这一步有提示：
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.26.243.110:6443 --token 0i1ato.5kroy56ed2vkr5lx \
    --discovery-token-ca-cert-hash sha256:7f9fda6c143d5cf9034616311581e43f7f1083574be032c759be10e06d783ed3
```

根据上面的提示创建config文件,并安装网络插件
网络插件可见:
https://kubernetes.io/docs/concepts/cluster-administration/addons/


我在这儿选择的是安装flannel
地址:https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml

可以把这个yaml文件download下来再去执行create
```
# kubectl create -f kube-flannel.yaml
podsecuritypolicy.extensions/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created
```

查看下pod的状态
```
[root@k8s-master ~]# kubectl get pod --all-namespaces
NAMESPACE     NAME                                 READY   STATUS     RESTARTS   AGE
kube-system   coredns-d5947d4b-c2ptm               0/1     Pending    0          12m
kube-system   coredns-d5947d4b-v6brp               0/1     Pending    0          12m
kube-system   etcd-k8s-master                      1/1     Running    0          11m
kube-system   kube-apiserver-k8s-master            1/1     Running    0          11m
kube-system   kube-controller-manager-k8s-master   1/1     Running    0          11m
kube-system   kube-flannel-ds-amd64-mqvf6          0/1     Init:0/1   0          3m31s
kube-system   kube-proxy-mxmhw                     1/1     Running    0          12m
kube-system   kube-scheduler-k8s-master            1/1     Running    0          11m
[root@k8s-master ~]# kubectl describe pod kube-flannel-ds-amd64-mqvf6 -n kube-system
Name:               kube-flannel-ds-amd64-mqvf6
Namespace:          kube-system
Priority:           0
PriorityClassName:  <none>
Node:               k8s-master/172.26.243.110
Start Time:         Thu, 25 Apr 2019 16:04:57 +0800
Labels:             app=flannel
                    controller-revision-hash=8676477c4
                    pod-template-generation=1
                    tier=node
Annotations:        <none>
Status:             Pending
IP:                 172.26.243.110
Controlled By:      DaemonSet/kube-flannel-ds-amd64
Init Containers:
  install-cni:
    Container ID:
    Image:         quay.io/coreos/flannel:v0.11.0-amd64
    Image ID:
    Port:          <none>
    Host Port:     <none>
    Command:
      cp
    Args:
      -f
      /etc/kube-flannel/cni-conf.json
      /etc/cni/net.d/10-flannel.conflist
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /etc/cni/net.d from cni (rw)
      /etc/kube-flannel/ from flannel-cfg (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from flannel-token-p8ccb (ro)
Containers:
  kube-flannel:
    Container ID:
    Image:         quay.io/coreos/flannel:v0.11.0-amd64
    Image ID:
    Port:          <none>
    Host Port:     <none>
    Command:
      /opt/bin/flanneld
    Args:
      --ip-masq
      --kube-subnet-mgr
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Limits:
      cpu:     100m
      memory:  50Mi
    Requests:
      cpu:     100m
      memory:  50Mi
    Environment:
      POD_NAME:       kube-flannel-ds-amd64-mqvf6 (v1:metadata.name)
      POD_NAMESPACE:  kube-system (v1:metadata.namespace)
    Mounts:
      /etc/kube-flannel/ from flannel-cfg (rw)
      /run/flannel from run (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from flannel-token-p8ccb (ro)
Conditions:
  Type              Status
  Initialized       False
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  run:
    Type:          HostPath (bare host directory volume)
    Path:          /run/flannel
    HostPathType:
  cni:
    Type:          HostPath (bare host directory volume)
    Path:          /etc/cni/net.d
    HostPathType:
  flannel-cfg:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      kube-flannel-cfg
    Optional:  false
  flannel-token-p8ccb:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  flannel-token-p8ccb
    Optional:    false
QoS Class:       Guaranteed
Node-Selectors:  beta.kubernetes.io/arch=amd64
Tolerations:     :NoSchedule
                 node.kubernetes.io/disk-pressure:NoSchedule
                 node.kubernetes.io/memory-pressure:NoSchedule
                 node.kubernetes.io/network-unavailable:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute
                 node.kubernetes.io/pid-pressure:NoSchedule
                 node.kubernetes.io/unreachable:NoExecute
                 node.kubernetes.io/unschedulable:NoSchedule
Events:
  Type    Reason     Age    From                 Message
  ----    ------     ----   ----                 -------
  Normal  Scheduled  3m46s  default-scheduler    Successfully assigned kube-system/kube-flannel-ds-amd64-mqvf6 to k8s-master
  Normal  Pulling    3m46s  kubelet, k8s-master  Pulling image "quay.io/coreos/flannel:v0.11.0-amd64"

```


发现用了的镜像是`quay.io/coreos/flannel:v0.11.0-amd64`,因为墙的原因,可以考虑先下载这个镜像load到节点去,再来执行创建flannel插件的步骤。

参考:
https://kubernetes.io/zh/docs/setup/independent/install-kubeadm/
https://kubernetes.io/docs/setup/independent/install-kubeadm/





## 添加节点node
先把flannel插件对应的镜像load到node节点上
```
[root@k8s-node1 tmp]# docker load < flannel.tar
7bff100f35cb: Loading layer [==================================================>]  4.672MB/4.672MB
5d3f68f6da8f: Loading layer [==================================================>]  9.526MB/9.526MB
9b48060f404d: Loading layer [==================================================>]  5.912MB/5.912MB
3f3a4ce2b719: Loading layer [==================================================>]  35.25MB/35.25MB
9ce0bb155166: Loading layer [==================================================>]   5.12kB/5.12kB
Loaded image: quay.io/coreos/flannel:v0.11.0-amd64
[root@k8s-node1 tmp]# docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
quay.io/coreos/flannel   v0.11.0-amd64       ff281650a721        2 months ago        52.6MB

```

根据init master后的输出来添加节点
```
[root@k8s-node1 tmp]# kubeadm join 172.26.243.110:6443 --token 0i1ato.5kroy56ed2vkr5lx \
>     --discovery-token-ca-cert-hash sha256:7f9fda6c143d5cf9034616311581e43f7f1083574be032c759be10e06d783ed3
[preflight] Running pre-flight checks
	[WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri
/	[WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.14" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

[root@k8s-node1 tmp]#
[root@k8s-node1 tmp]# kubectl get nodes
The connection to the server localhost:8080 was refused - did you specify the right host or port?
[root@k8s-node1 tmp]#
[root@k8s-node1 tmp]#
[root@k8s-node1 tmp]# mkdir -p $HOME/.kube
[root@k8s-node1 tmp]# scp -i k8s-master:/etc/kubernetes/admin.conf $HOME/.kube/config
usage: scp [-12346BCpqrv] [-c cipher] [-F ssh_config] [-i identity_file]
           [-l limit] [-o ssh_option] [-P port] [-S program]
           [[user@]host1:]file1 ... [[user@]host2:]file2
[root@k8s-node1 tmp]# scp -i k8s-master:/etc/kubernetes/admin.conf $HOME/.kube/config^C
[root@k8s-node1 tmp]#
[root@k8s-node1 tmp]# scp k8s-master:/etc/kubernetes/admin.conf $HOME/.kube/config
The authenticity of host 'k8s-master (172.26.243.110)' can't be established.
ECDSA key fingerprint is SHA256:ABQQSCI0KxtjuCL5a3nNRf+bdNmoPLxYXUJVRtlqf3w.
ECDSA key fingerprint is MD5:dd:cb:ed:d5:d6:cd:50:ec:c2:bb:65:6f:d7:3f:6f:4a.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'k8s-master,172.26.243.110' (ECDSA) to the list of known hosts.
root@k8s-master's password:
admin.conf                                                                                                                                                 100% 5454    12.1MB/s   00:00
[root@k8s-node1 tmp]#
[root@k8s-node1 tmp]# chown $(id -u):$(id -g) $HOME/.kube/config
[root@k8s-node1 tmp]# ls
Aegis-<Guid(5A2C30A2-A87D-490A-9281-6765EDAD7CBA)>  flannel.tar  systemd-private-603a83b0fba9490f8c053d023082ac11-chronyd.service-8KhMeD
[root@k8s-node1 tmp]# kubectl get node
NAME         STATUS   ROLES    AGE    VERSION
k8s-master   Ready    master   25m    v1.14.1
k8s-node1    Ready    <none>   119s   v1.14.1
```
