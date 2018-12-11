<!-- toc -->
## 要保证有rr模块
```
[root@node1 flannel]#  lsmod | grep ip_vs
ip_vs_sh               12688  0
ip_vs_wrr              12697  0
ip_vs_rr               12600  13
ip_vs                 141092  19 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          133387  7 ip_vs,nf_nat,nf_nat_ipv4,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_netlink,nf_conntrack_ipv4
libcrc32c              12644  3 ip_vs,nf_nat,nf_conntrack
```

没开启加载方式：
```
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
```

## 启用 ipvs 后与 1.7 版本的配置差异如下：
* 增加 `–feature-gates=SupportIPVSProxyMode=true` 选项，用于告诉 kube-proxy 开启 ipvs 支持，目前 ipvs 默认未开启
* 增加 `ipvs-min-sync-period`、`–ipvs-sync-period`、`–ipvs-scheduler` 三个参数用于调整 ipvs，具体参数值请自行查阅 ipvs 文档
* 增加 –masquerade-all 选项，以确保反向流量通过
* 打开 ipvs 需要安装 ipvsadm 软件， 在 node 中安装
    ```
    yum install ipvsadm -y
    ipvsadm -L -n
    ```

重点说一下 –masquerade-all 选项:
kube-proxy ipvs 是基于 NAT 实现的，当创建一个 service 后，kubernetes 会在每个节点上创建一个网卡，同时帮你将 Service IP(VIP) 绑定上，此时相当于每个 Node 都是一个 ds，而其他任何 Node 上的 Pod，甚至是宿主机服务(比如 kube-apiserver 的 6443)都可能成为 rs；
按照正常的 lvs nat 模型，所有 rs 应该将 ds 设置成为默认网关，以便数据包在返回时能被 ds 正确修改；
在 kubernetes 将 vip 设置到每个 Node 后，默认路由显然不可行，所以要设置 –masquerade-all 选项，以便反向数据包能通过。

注意：`--masquerade-all` 选项与 Calico 安全策略控制不兼容,请酌情使用
