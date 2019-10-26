<!-- toc -->
## 命令补全
```
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
```

### 添加ns的快捷方式
```
git clone https://github.com/ahmetb/kubectx /opt/kubectx
ln -s /opt/kubectx/kubens /usr/local/bin/ns
```

### 去掉master的污点
```
kubectl taint nodes master node-role.kubernetes.io/master-
```

### 将svc的type类型从ClusterIP变为NodePort
```
kubectl patch service istio-ingressgateway  -p '{"spec":{"type":"NodePort"}}' -n istio-system

kubectl patch service alertmanager-main   -p '{"spec":{"type":"NodePort"}}' -n monitoring

kubectl patch service prometheus-k8s    -p '{"spec":{"type":"NodePort"}}' -n monitoring

kubectl patch service grafana   -p '{"spec":{"type":"NodePort"}}' -n monitoring
```
