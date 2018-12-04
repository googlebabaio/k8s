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