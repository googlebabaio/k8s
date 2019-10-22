参考：
[Etcd 架构与实现解析](http://jolestar.com/etcd-architecture/)
[Etcd解析](https://jimmysong.io/kubernetes-handbook/concepts/etcd.html)


查看ETCD的状态
```
etcdctl --endpoints=https://172.18.53.221:2379,https://172.18.53.223:2379,https://172.18.53.224:2379 --ca-file=/etc/kubernetes/ssl/ca.pem --cert-file=/etc/kubernetes/ssl/etcd.pem --key-file=/etc/kubernetes/ssl/etcd-key.pem ls /kubernetes/network/subnets
```

创建网络配置
```
/etc/kubernetes/bin/etcdctl --ca-file /etc/kubernetes/ssl/ca.pem --cert-file /etc/kubernetes/ssl/flanneld.pem --key-file /etc/kubernetes/ssl/flanneld-key.pem \
--no-sync -C https://172.18.53.221:2379,https://172.18.53.223:2379,https://172.18.53.224:2379 \
mk /kubernetes/network/config '{ "Network": "10.2.0.0/16", "Backend": { "Type": "vxlan", "VNI": 1 }}'
```
