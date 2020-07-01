# kata的安装
## 配置源
```
curl -O etc/yum.repos.d/CentOS-Base.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo

curl -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```

## wget下载kata相应的rpm包
```
wget -r --no-parent  http://download.opensuse.org/repositories/home:/katacontainers:/releases:/x86_64:/master/CentOS_7/x86_64/
```

## 安装
```
[root@ccsk8s3 kata]# rpm -ivh *
warning: kata-containers-image-1.11.0~rc0-45.2.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID 6063f3ed: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:qemu-vanilla-data-4.1.1+git.99c58################################# [  8%]
   2:qemu-vanilla-bin-4.1.1+git.99c587################################# [ 17%]
   3:qemu-vanilla-4.1.1+git.99c5874a9b################################# [ 25%]
   4:kata-shim-bin-1.11.0~rc0-44.2    ################################# [ 33%]
   5:kata-shim-1.11.0~rc0-44.2        ################################# [ 42%]
   6:kata-proxy-bin-1.11.0~rc0-46.2   ################################# [ 50%]
   7:kata-proxy-1.11.0~rc0-46.2       ################################# [ 58%]
   8:kata-linux-container-5.4.32.73-62################################# [ 67%]
   9:kata-ksm-throttler-1.11.0~rc0-50.################################# [ 75%]
  10:kata-containers-image-1.11.0~rc0-################################# [ 83%]
  11:kata-runtime-1.11.0~rc0-67.2     ################################# [ 92%]
  12:kata-linux-container-debug-5.4.32################################# [100%]
```

## 查看版本
```
[root@ccsk8s3 kata]# kata-runtime --version
kata-runtime  : 1.11.0-rc0
   commit   : f7f5d42390b15b416f198570d7778dc09725a1d0
   OCI specs: 1.0.1-dev
```

## 检查虚拟化环境
因为 kata 实际上会创建虚拟机，所以要求安装 kata 的主机开启 nested virtualization：
```
[root@ccsk8s3 kata]# kata-runtime kata-check
System is capable of running Kata Containers
System can currently create Kata Containers
```

# 和docker结合使用
这个和docker的架构演进有关，docker和containerd的关系是：
docker daemon --> contained--> runc

而kata是符合oci标注的，所以runc可以变成runv。故而只需要将containerd最后调用的runc替换为runv即可。

```
dir=/etc/systemd/system/docker.service.d
file="$dir/kata-containers.conf"
mkdir -p "$dir"
test -e "$file" || echo -e "[Service]\nType=simple\nExecStart=\nExecStart=/usr/bin/dockerd -D --default-runtime runc" | sudo tee "$file"
grep -q "kata-runtime=" $file || sudo sed -i 's!^\(ExecStart=[^$].*$\)!\1 --add-runtime kata-runtime=/usr/bin/kata-runtime!g' "$file"
```


## 重启后查看docker的状态
重启后，可以看到在`Runtimes`除了runc外多了一个kata-runtime

```
[root@ccsk8s3 kata]# docker info
Client:
 Debug Mode: false

Server:
 Containers: 17
  Running: 0
  Paused: 0
  Stopped: 17
 Images: 21
 Server Version: 19.03.8
 Storage Driver: overlay2
  Backing Filesystem: <unknown>
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: kata-runtime runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 7ad184331fa3e55e52b890ea95e65ba581ae3429
 runc version: dc9208a3303feef5b3839f4323d9beb36df0a9dd
 init version: fec3683
 Security Options:
  seccomp
   Profile: default
 Kernel Version: 3.10.0-693.el7.x86_64
 Operating System: CentOS Linux 7 (Core)
 OSType: linux
 Architecture: x86_64
 CPUs: 12
 Total Memory: 7.449GiB
 Name: ccsk8s3
 ID: ZCYJ:5LIY:67PU:NUZE:43FZ:IRJG:CH3K:PWEE:K7EZ:E5PU:GLZ2:A7LI
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false
 Product License: Community Engine
```

## 创建容器
因为runc和runv是可以并存的，所以我们只需要指定`--runtime`就可以使用不同的运行时

```
docker run -d --name kata-test -p 8081:80 --runtime kata-runtime nginx
docker run -d --name runc-test -p 8082:80 nginx
```

> 这儿run的kata的时候可能会有个问题：就是调用kata-runtime的时候，调用的路径是`/usr/local/bin/kata-runtime`，而实际的路径在`/usr/local/bin/kata-runtime`，会报找不到的错误，只需要做一个软链接即可。

创建完成并测试：
```
[root@ccsk8s3 ~]#  docker run -d --name kata-test -p 8081:80 --runtime kata-runtime nginx
4ff16f4992d27edcf790f8d7d51d3d106cbbbde2208313451f1358a8d8de39f8
[root@ccsk8s3 ~]#
[root@ccsk8s3 ~]#
[root@ccsk8s3 ~]#
[root@ccsk8s3 ~]#  docker run -d --name runc-test -p 8082:80 nginx
b466722af2e858ff5963167a58fddaecd9afc8ed972201628f97fcbc6f2e2d8c
[root@ccsk8s3 ~]#
[root@ccsk8s3 ~]# docker ps |grep test
b466722af2e8        nginx                  "/docker-entrypoint.…"   About a minute ago   Up About a minute   0.0.0.0:8082->80/tcp   runc-test
4ff16f4992d2        nginx                  "/docker-entrypoint.…"   About a minute ago   Up About a minute   0.0.0.0:8081->80/tcp   kata-test
[root@ccsk8s3 ~]#
[root@ccsk8s3 ~]# curl localhost:8081
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[root@ccsk8s3 ~]# curl localhost:8082
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

也可以用` kata-runtime list`来看kata的runtime
```
[root@ccsk8s3 ~]# kata-runtime list
ID                                                                 PID         STATUS      BUNDLE                                                                                                                               CREATED                          OWNER
4ff16f4992d27edcf790f8d7d51d3d106cbbbde2208313451f1358a8d8de39f8   3176        running     /run/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/4ff16f4992d27edcf790f8d7d51d3d106cbbbde2208313451f1358a8d8de39f8   2020-06-30T02:40:38.099200196Z   #0
[root@ccsk8s3 ~]#
```



# 结合kubernetes使用
官网对结合k8s的使用都是从kata+初始化安装k8s开始的。但真实的情况一般是，我们想拿已经存在的k8s集群的某个节点加入kata来做一些验证，所以下面的步骤真实有效：

1. 以前是docker，所以现在使用containerd，这个涉及到docker调用containerd和k8s调用cri再调用containerd（containerd1.1集成了cri插件，所以一切才会变得更简单）
2. 配置containerd，因为上面已经安装了kata
3. docker可以不启动，和架构有关的。

## container的service文件
让containerd加入到systemctl管理

vi /usr/lib/systemd/system/contained.service

```
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
```

systemctl enable containerd
systemctl restart containerd
systemctl status containerd


## 生成containerd的配置
```
containerd --version
mkdir -p /etc/containerd/
containerd config default > /etc/containerd/config.toml
```

## 修改配置，加上runtime.kata
```
 [plugins.cri.containerd]
     # snapshotter = "overlayfs"
      no_pivot = false
      [plugins.cri.containerd.runtimes]
        [plugins.cri.containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v1"
        [plugins.cri.containerd.runtimes.runc.options]
          NoPivotRoot = false
          NoNewKeyring = false
          ShimCgroup = ""
          IoUid = 0
          IoGid = 0
          BinaryName = "runc"
          Root = ""
          CriuPath = ""
          SystemdCgroup = false
        [plugins.cri.containerd.runtimes.kata]
          runtime_type = "io.containerd.kata.v2"
        [plugins.cri.containerd.runtimes.katacli]
          runtime_type = "io.containerd.runc.v1"
        [plugins.cri.containerd.runtimes.katacli.options]
          NoPivotRoot = false
          NoNewKeyring = false
          ShimCgroup = ""
          IoUid = 0
          IoGid = 0
          BinaryName = "/usr/bin/kata-runtime"
          Root = ""
          CriuPath = ""
          SystemdCgroup = false
```

> 需要注意一个sandbox_image涉及到kubelet启动pause的版本的地方哦！

## 配置kubelet的启动项，让默认的docker.sock变成containerd.sock
/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf

添加：
Environment="KUBELET_EXTRA_ARGS=--container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"

这个地方有个坑哦，在`10-kubeadm.conf`里面有两个地方有KUBELET_EXTRA_ARGS，所以注意看是否生效了的。

## 创建 runtimeclass

```
apiVersion: node.k8s.io/v1beta1
kind: RuntimeClass
metadata:
  name: kata
handler: kata #这个名字和config.toml中的配置应该一样
```


## 给调度的节点打上label
```
kubectl get nodes --show-labels
kubectl label node ccsk8s3 runtimetype=kata
kubectl get nodes --show-labels
```

## cri
```
cri导入镜像命令(cri导入镜像)：

   ctr cri load  images.tar
containerd导入镜像命令(containerd导入镜像)：

   ctr images import images.tar
```


## 创建测试pod
```
apiVersion: v1
kind: Pod
metadata:
  name: kata-nginx
spec:
  runtimeClassName: kata
  containers:
    - name: nginx
      image: nginx
      ports:
      - containerPort: 80
  nodeSelector:
    runtimetype: kata
```

## 查看结果
```
[root@ccsk8s3 config]# uname -r
3.10.0-693.el7.x86_64
[root@ccsk8s3 config]#
[root@ccsk8s3 config]#  kubectl get pod -owide
NAME                                      READY   STATUS    RESTARTS   AGE    IP            NODE      NOMINATED NODE   READINESS GATES
kata-nginx                                1/1     Running   0          22m    10.100.9.75   ccsk8s3   <none>           <none>
nfs-client-provisioner-84d79fdd45-ds7vv   1/1     Running   0          102m   10.100.3.71   ccsk8s2   <none>           <none>
[root@ccsk8s3 config]#
[root@ccsk8s3 config]#
[root@ccsk8s3 config]# kubectl exec  -it kata-nginx -- uname -r
5.4.32-62.2.container
[root@ccsk8s3 config]#
```

```shell
[root@ccsk8s3 config]# kata-runtime list
ID                                                                 PID         STATUS      BUNDLE                                                                                                                  CREATED                          OWNER
3cf16f9bcfac06ef7ebf99df22ea46fa254540675392e5f7ebd5d4abe6c46fb5   -1          running     /run/containerd/io.containerd.runtime.v2.task/k8s.io/3cf16f9bcfac06ef7ebf99df22ea46fa254540675392e5f7ebd5d4abe6c46fb5   2020-06-30T08:37:49.30253615Z    #0
e55a1e6a927cb1d75fc53fac98f78b745f7ef6f16503545b4ee0232c7c696882   -1          running     /run/containerd/io.containerd.runtime.v2.task/k8s.io/e55a1e6a927cb1d75fc53fac98f78b745f7ef6f16503545b4ee0232c7c696882   2020-06-30T08:48:04.432246813Z   #0
[root@ccsk8s3 config]#
[root@ccsk8s3 config]#
[root@ccsk8s3 config]# crictl ps
CONTAINER ID        IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID
9e387b4008562       8a21fea074d71       4 minutes ago       Running             calico-node         0                   6e5a386829dc6
e55a1e6a927cb       2622e6cca7ebb       13 minutes ago      Running             nginx               0                   3cf16f9bcfac0
[root@ccsk8s3 config]#
[root@ccsk8s3 config]#
[root@ccsk8s3 config]# crictl images
IMAGE                                 TAG                 IMAGE ID            SIZE
docker.io/calico/cni                  v3.9.5              5d294503bbba8       58MB
docker.io/calico/node                 v3.9.5              8a21fea074d71       88.8MB
docker.io/calico/pod2daemon-flexvol   v3.9.5              35001c355868d       4.91MB
docker.io/library/nginx               latest              2622e6cca7ebb       136MB
k8s.gcr.io/pause                      3.2                 80d28bedfe5de       686kB
[root@ccsk8s3 config]#
[root@ccsk8s3 config]#

```


参考：

- https://zhuanlan.zhihu.com/p/109256949
- https://lingxiankong.github.io/2018-07-20-katacontainer-docker-k8s.html

- 打开新世界：https://containerd.io/docs/getting-started/
- https://github.com/containerd/cri/blob/master/docs/config.md


```go
package main

import (
        "context"
        "log"

        "github.com/containerd/containerd"
        "github.com/containerd/containerd/oci"
        "github.com/containerd/containerd/namespaces"
)

func main() {
        if err := redisExample(); err != nil {
                log.Fatal(err)
        }
}

func redisExample() error {
        client, err := containerd.New("/run/containerd/containerd.sock")
        if err != nil {
                return err
        }
        defer client.Close()

        ctx := namespaces.WithNamespace(context.Background(), "example")
        image, err := client.Pull(ctx, "docker.io/library/redis:alpine", containerd.WithPullUnpack)
        if err != nil {
                return err
        }
        log.Printf("Successfully pulled %s image\n", image.Name())

        container, err := client.NewContainer(
                ctx,
                "redis-server",
                containerd.WithNewSnapshot("redis-server-snapshot", image),
                containerd.WithNewSpec(oci.WithImageConfig(image)),
        )
        if err != nil {
                return err
        }
        defer container.Delete(ctx, containerd.WithSnapshotCleanup)
        log.Printf("Successfully created container with ID %s and snapshot with ID redis-server-snapshot", container.ID())

        return nil
}
```
