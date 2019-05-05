## 1.规划

### 节点规划
|节点   | IP  | 角色  |部署软件|
|---|---|---|---|
| k8s-sc-harbor  |  192.168.3.9 | harbor仓库  |docker、docker-compose、harbor|
| k8s-sc-master  | 192.168.3.94  |  master |docker、kubelt、kubeadm、kubectl、helm|
| k8s-sc-node1  | 192.168.3.124  |  node1 |docker、kubelet、kubeadm|
| k8s-sc-node2  |  192.168.3.129| node2 |docker、kubelet、kubeadm|
| k8s-sc-node3  |  192.168.3.131|node3 |docker、kubelet、kubeadm|

### 目录规划
docker的根目录位置: /dockerdata
harbor的暴露端口: 8888
harbor的根目录: /harbordata

### 软件及版本
|软件名称 |软件版本 | 说明|
|---|---|---|
|harbor   |   | |
|docker | | |
|docker-compose   |   |   |
|kubelet | |  |
|kube-proxy | | |
|kube-apiserver | | |
|kube-schduler | |  |
|kube-controller | | |
|core-dns | | |
| flannel| | |

软件安装(基础的rpm包的安装)
```
```


2.安装docker(每台)
```

```

3.部署harbor

4.准备镜像

5.部署master

6.部署node

7.测试
