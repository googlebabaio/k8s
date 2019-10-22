
<!-- toc --> 

* * * * *
# 部署kubectl工具
## 1.准备二进制文件
```
[root@master temp]# cd /usr/local/src/kubernetes/client/bin/
[root@master bin]# cp kubectl /etc/kubernetes/bin/
[root@master bin]# scp kubectl node1:/etc/kubernetes/bin/
[root@master bin]# scp kubectl node2:/etc/kubernetes/bin/
```
## 2.创建 admin 证书签名请求
```
[root@master temp]# cat admin-csr.json 
{
  "CN": "admin",
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
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```

## 3.生成admin证书

```
[root@master temp]# cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
    -ca-key=/etc/kubernetes/ssl/ca-key.pem \
    -config=/etc/kubernetes/ssl/ca-config.json \
    -profile=kubernetes admin-csr.json | cfssljson -bare admin
2018/09/20 14:28:30 [INFO] generate received request
2018/09/20 14:28:30 [INFO] received CSR
2018/09/20 14:28:30 [INFO] generating key: rsa-2048
2018/09/20 14:28:31 [INFO] encoded CSR
2018/09/20 14:28:31 [INFO] signed certificate with serial number 56918806213694388959411509627148507078688254192
2018/09/20 14:28:31 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
[root@master temp]# ls -l admin*
-rw-r--r-- 1 root root 1009 Sep 20 14:28 admin.csr
-rw-r--r-- 1 root root  229 Sep 20 14:26 admin-csr.json
-rw------- 1 root root 1675 Sep 20 14:28 admin-key.pem
-rw-r--r-- 1 root root 1399 Sep 20 14:28 admin.pem
```

## 4.分发证书
只想给master节点使用，那就不需要再scp到node节点上去。
```
[root@master temp]# cp admin*.pem ..
```


## 5.配置kubectl的使用的config
因为rbac认证的原因，默认会内置一些角色，可以查看admin-crs.json中的`"O": "system:masters"`
### 5.1 设置集群参数
```
kubectl config set-cluster kubernetes \
    --certificate-authority=/etc/kubernetes/ssl/ca.pem \
    --embed-certs=true \
    --server=https://172.18.53.221:6443
```


### 5.2 设置客户端认证参数
```
kubectl config set-credentials admin \
   --client-certificate=/etc/kubernetes/ssl/admin.pem \
   --embed-certs=true \
   --client-key=/etc/kubernetes/ssl/admin-key.pem
```

### 5.3 设置上下文参数
```
kubectl config set-context kubernetes \
   --cluster=kubernetes \
   --user=admin
```

### 5.4 设置默认上下文
```
kubectl config use-context kubernetes
```

***以上所有的操作的最终目的是为了生成 ~/.kube/config文件***，如果node节点想使用kubectl的话，就需要将相应的`admin*.pem`拷贝到`/etc/kubernetes/ssl`目录，以及拷贝一份 `~/.kube/config`文件

## 6.使用kubectl查看当前的资源情况
```
[root@master ssl]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}  
```
