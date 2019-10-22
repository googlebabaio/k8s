static pod 由kubelet来进行启动，apiserver是不能对其管理的。这样就可以解决一个蛋生鸡还是鸡生蛋的问题，也让组件容器化变为可能。

配置方法
在kubelet的配置文件加上相应的`pod-manifest-path=/opt/kubernetes/manifest`选项
```
[root@node1 manifest]# cat  /usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/opt/kubernetes/bin/kubelet \
  --address=192.168.3.28 \
  --hostname-override=192.168.3.28 \
  --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.0 \
  --experimental-bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
  --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
  --cert-dir=/opt/kubernetes/ssl \
  --network-plugin=cni \
  --cni-conf-dir=/etc/cni/net.d \
  --cni-bin-dir=/opt/kubernetes/bin/cni \
  --cluster-dns=10.1.0.2 \
  --cluster-domain=cluster.local. \
  --hairpin-mode hairpin-veth \
  --allow-privileged=true \
  --fail-swap-on=false \
  --logtostderr=true \
  --v=2 \
  --log-dir=/opt/kubernetes/log \
  --pod-manifest-path=/opt/kubernetes/manifest
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target


[root@node1 manifest]# ps -ef | grep kubelet
root     31443     1 14 17:47 ?        00:00:02 /opt/kubernetes/bin/kubelet --address=192.168.3.28 --hostname-override=192.168.3.28 --pod-infra-container-image=mirrorgooglecontainers/pause-
amd64:3.0 --experimental-bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig --cert-dir=/opt/kubernetes/ssl --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/kubernetes/bin/cni --cluster-dns=10.1.0.2 --cluster-domain=cluster.local. --hairpin-mode hairpin-veth --allow-privileged=true --fail-swap-on=false --logtostderr=true --v=2 --log-dir=/opt/kubernetes/log --pod-manifest-path=/opt/kubernetes/manifest
```

然后在/opt/kubernetes/manifest加上相应的pod的yaml文件即可
```
[root@node1 manifest]# ls
static-web.yaml
[root@node1 manifest]# cat static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    role: myrole
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
          protocol: TCP
```

过一下,就可以在pod的列表中看到
```
[root@node1 manifest]# kubectl get pod | grep static
static-web-192.168.3.28                     1/1       Running     0          17m
```
