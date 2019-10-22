<!-- toc -->

# 概述
在k8s1.14之后,kubeadm安装已经快要成为事实的标准,在之前参加的CKA考试中,发现部署的环境以及是使用kubeadm部署的了，所以对kubeadm的使用也必需要有所了解了！

在安装完master节点后，最后会出现提示的信息，告知节点怎么添加，如下

```
 kubeadm join 172.26.243.110:6443 --token 5e01an.651vofaq7w7b7h3a \
 --discovery-token-ca-cert-hash sha256:4ab1554087ab674b0ec716a4eafc28211fb8e4d5338ffa33f896e2984e929054
```

## `kubeadm join`一般有3个选项
- 172.26.243.110:6443 这个是apiserver的地址，需要按照实际的情况来填
- token: 一般为24小时有效的令牌可以使用添加节点的信息(也可以创建永不过期的token)
- discovery-token-ca-cert-hash : ca证书sha256编码

## 选项`--token`
token的查看
```
kubeadm token list
```

token的创建
```
kubeadm token create
```

永久token的创建
```
# kubeadm token create --ttl 0
b9y0pe.t854qjr9kqk8vt6r

# kubeadm token list | grep forever
b9y0pe.t854qjr9kqk8vt6r   <forever>   <never>   authentication,signing   <none>  system:bootstrappers:kubeadm:default-node-token
```

## 选项`--discovery-token-ca-cert-hash`
ca证书sha256编码的获取方式:
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

# 在忘记token的情况下,node节点加入k8s集群的步骤
- 1.获取apiserver的信息  ${apiserver:port}
- 2.创建或者获取一个token ${token}
- 3.获取ca证书的sha256编码 ${token-ca-cert-hash}
- 4.组装命令
  ```
  kubeadm join ${apiserver:port} --token ${token} --discovery-token-ca-cert-hash ${token-ca-cert-hash}
  ```
