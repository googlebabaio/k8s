## 官方提供的几种Kubernetes部署方式
### minikube
Minikube是一个工具，可以在本地快速运行一个单点的Kubernetes，尝试Kubernetes或日常开发的用户使用。不能用于生产环境。

官方地址：https://kubernetes.io/docs/setup/minikube/

### kubeadm
Kubeadm也是一个工具，提供kubeadm init和kubeadm join，用于快速部署Kubernetes集群。

官方地址：https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/

### 二进制包
从官方下载发行版的二进制包，手动部署每个组件，组成Kubernetes集群。
