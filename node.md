节点（上图橘色方框）是物理或者虚拟机器，作为Kubernetes worker，通常称为Minion。每个节点都运行如下Kubernetes关键组件：
Kubelet：是主节点代理。
Kube-proxy：Service使用其将链接路由到Pod，如上文所述。
Docker或Rocket：Kubernetes使用的容器技术来创建容器。