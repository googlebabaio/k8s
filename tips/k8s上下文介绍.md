<!-- toc -->

在有证书的机器上
## 1.生成新的kubeconfig文件
```
kubectl config set-credentials devuser \
--client-certificate=/opt/kubernetes/ssl/devuser.pem \
--client-key=/opt/kubernetes/ssl/devuser-key.pem \
--embed-certs=true \
--kubeconfig=kubeconfig
```

> 特别注意
> embed-certs选项，如果为false，代表里面的秘钥会以路径的方式生成，这个对于非本地的config是不可取的

## 2.创建集群入口
```
kubectl config set-cluster k8s --server=https://172.18.53.221:6443 --kubeconfig=kubeconfig
```

## 3.指定集群证书文件
```
kubectl config set-cluster k8s --certificate-authority=/opt/kubernetes/ssl/ca.pem --embed-certs=true --kubeconfig=kubeconfig
```

kubernetes.pem  是集群的证书，有些地方应该是ca
要参考apiserver的配置文件的client-ca-file 一项
```
[Service]
ExecStart=/opt/kubernetes/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \
  --bind-address=172.18.53.221 \
  --insecure-bind-address=0.0.0.0 \
  --insecure-port=8080 \
  --authorization-mode=Node,RBAC \
  --runtime-config=rbac.authorization.k8s.io/v1 \
  --kubelet-https=true \
  --anonymous-auth=false \
  --basic-auth-file=/opt/kubernetes/ssl/basic-auth.csv \
  --enable-bootstrap-token-auth \
  --token-auth-file=/opt/kubernetes/ssl/bootstrap-token.csv \
  --service-cluster-ip-range=10.1.0.0/16 \
  --service-node-port-range=20000-40000 \
  --tls-cert-file=/opt/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/opt/kubernetes/ssl/kubernetes-key.pem \
  --client-ca-file=/opt/kubernetes/ssl/ca.pem \
  --service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --etcd-cafile=/opt/kubernetes/ssl/ca.pem \
  --etcd-certfile=/opt/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/opt/kubernetes/ssl/kubernetes-key.pem \
  --etcd-servers=https://172.18.53.221:2379,https://172.18.53.223:2379,https://172.18.53.224:2379 \
  --enable-swagger-ui=true \
  --allow-privileged=true \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/opt/kubernetes/log/api-audit.log \
  --event-ttl=1h \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log \
  --enable-swagger-ui=true
```

各种ca，tls，请参考：[https://blog.csdn.net/qq_34463875/article/details/78042852](https://blog.csdn.net/qq_34463875/article/details/78042852)

## 4.创建环境配置文件context
```
kubectl config set-context k8s \
--cluster=k8s \
--user=devuser \
--namespace=kube-system \
--kubeconfig=kubeconfig
```

注意：
可以加上namespace作为默认的namespace是选项

## 5.将kubeconfig文件 copy到 /root/.kube/config
mv kubeconfig /root/.kube/config

## 6.激活上下文
```
kubectl config use-context k8s
```
切换之后看到的新的上下文应该为如下：
```
[root@nazeebodan ~]# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: https://172.18.53.221:6443
  name: k8s
contexts:
- context:
    cluster: k8s
    namespace: kube-system
    user: devuser
  name: k8s
current-context: k8s
kind: Config
preferences: {}
users:
- name: devuser
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED

```

7、检查kubectl get pod


========================================
![](images/screenshot_1532850467585.png)
登录dashboard的时候，会有2个选项

```
	3.1获取sa的名字
	kubectl get secret -n kube-system|grep admin-user
	3.2 获取密钥
	kubectl describe secret admin-user-token-ml8d2 -n kube-system


```

将获取到的密码注入到config文件
```
kubectl config set-credentials admin --token=eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTZoZnhoIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJhOGJiNmIwNC05MjBkLTExZTgtODc2MS0wMDE2M2UwNDYzYTciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.EZF855UNZfr7k2RQtZ5p2f-1bkn9C-EEBdk7Oh0ss08wQnxE4Y2FqZDfzU9YkT87XDYMxh_OKLWqSMPzw6ssGlQNFOQL8h6Y3IfkE4Q2gExzvQ8O9Ilqgj6o-9oRpE7R_GDY8x0OCSh6IwZWE7WSfaelpzVTLKfZpSafQJGU9FcdW-7pxD1dwo1bognoSlg6wVlLcaH3Tk0JahANsZNFnxp5dgTla1ijjgA9x5VDDo59YBWRZawArcoI3ha1DNxSfjb9ylB1VkJWbKT7x7DwG0KAhqhlb6lMvBFxJyK-PPihtPm69oCvQuctiMjAlRu5En0OCwsxbp5Xum8Q4ZPuEQ --kubeconfig=config.bak
```

将集群入口信息写入到config
```
kubectl config set-cluster kubernetes --server=https://172.18.53.221:6443 --kubeconfig=config.bak
```

将认证信息写入到config
```
kubectl config set-cluster kubernetes --certificate-authority=/opt/kubernetes/ssl/ca.pem --embed-certs=true --
kubeconfig=config.bak
```

创建环境配置文件
```
kubectl config set-context kubernetes --cluster=kubernetes --user=admin  --kubeconfig=config.bak
```

激活上下文
```
kubectl config use-context kubernetes --kubeconfig=config.bak
```




* * * * *

题外话：
1、创建serviceaccount
2、将角色绑定到serviceaccout
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

理一下user和sa的关系
1.
```
通过ca创建user-->生成证书-->创建角色(资源+操作)-->创建绑定(绑定里面就有目标用户了)-->绑定完成后就有相应的权限了

给user生成新的kubeconfig文件
创建集群入口
指定集群证书文件
创建环境配置文件(指定user和namespace)
激活上下文
```

2.如果想让user访问某pod
```
如果要让user访问某pod，那么pod有相应的sa，如果有特殊需求，那么先去建立，如下：
创建secret-->创建sa-->绑定pod ==>要访问这个pod说明就需要有相应的sa对应的token

接着
将获取到的密码注入到config文件，步骤：
给user生成新的kubeconfig文件
创建集群入口
指定集群证书文件
创建环境配置文件(指定user和namespace)
激活上下文，拿新生产的kubeconfig就可以用了。
```
