## 查看etcd的相关信息

```
# etcdctl --endpoints=https://192.168.3.6:2379,https://192.168.3.25:2379,https://192.168.3.26:2379,https://192.168.3.22:2379 \
>   --ca-file=/etc/kubernetes/ssl/ca.pem \
>   --cert-file=/etc/kubernetes/ssl/etcd.pem \
>   --key-file=/etc/kubernetes/ssl/etcd-key.pem member list

bb39f465356b6d: name=etcd-master peerURLs=https://192.168.3.6:2380 clientURLs=https://192.168.3.6:2379 isLeader=true
a1e30db0d38d29cf: name=etcd-node1 peerURLs=https://192.168.3.25:2380 clientURLs=https://192.168.3.25:2379 isLeader=false
c942a25792249cfa: name=etcd-node3 peerURLs=https://192.168.3.22:2380 clientURLs=https://192.168.3.22:2379 isLeader=false
dd3bdd7be2775880: name=etcd-node2 peerURLs=https://192.168.3.26:2380 clientURLs=https://192.168.3.26:2379 isLeader=false
```

## etcd备份
```
ETCDCTL_API=3 etcdctl --endpoints=https://192.168.3.6:2379,https://192.168.3.25:2379,https://192.168.3.26:2379,https://192.168.3.22:2379 \
  --cacert=/etc/kubernetes/ssl/ca.pem \
  --cert=/etc/kubernetes/ssl/etcd.pem \
  --key=/etc/kubernetes/ssl/etcd-key.pem snapshot save snapshotdb
```

## 查看etcd的备份信息
```
# ETCDCTL_API=3 etcdctl --endpoints=https://192.168.3.6:2379,https://192.168.3.25:2379,https://192.168.3.26:2379,https://192.168.3.22:2379 \
>   --cacert=/etc/kubernetes/ssl/ca.pem \
>   --cert=/etc/kubernetes/ssl/etcd.pem \
>   --key=/etc/kubernetes/ssl/etcd-key.pem --write-out=table snapshot status snapshotdb
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| a44716fc | 28376050 |       1106 |     3.7 MB |
+----------+----------+------------+------------+
```

## etcd 删除namespace
```
ETCDCTL_API=3 etcdctl --endpoints=https://192.168.3.15:2379 --cert=/etc/kubernetes/pki/etcd/server.crt  --key=/etc/kubernetes/pki/etcd/server.key --cacert=/etc/kubernetes/pki/etcd/ca.crt del /registry/namespaces/NAMESPACE_NAME_XXX
```


## 查看所有的key
```
#!/bin/bash
# Get kubernetes keys from etcd
export ETCDCTL_API=3
keys=`etcdctl --endpoints=https://192.168.3.15:2379 --cert=/etc/kubernetes/pki/etcd/server.crt  --key=/etc/kubernetes/pki/etcd/server.key --cacert=/etc/kubernetes/pki/etcd/ca.crt get /registry --prefix -w json|python -m json.tool|grep key|cut -d ":" -f2|tr -d '"'|tr -d ","`
for x in $keys;do
  echo $x|base64 -d|sort
done
```
