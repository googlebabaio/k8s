node节点pod无法启动/节点删除网络重置
node1之前反复添加过,添加之前需要清除下网络
```
# journalctl -u kubelet
 failed: rpc error: code = Unknown desc = NetworkPlugin cni failed to set up pod "nginx-8586cf59-rm4sh_default" network: failed to set bridge addr: "cni0" already has an IP address different from 10.244.2.1/24
12252 cni.go:227] Error while adding to cni network: failed to set bridge addr: "cni0" already
```
pod创建报错
```
kubectl describe pod nginx-test-5d6675ddb4-9j6vg

 Warning  FailedCreatePodSandBox  14m                    kubelet, host-192-168-3-94  Failed create pod sandbox: rpc error: code = Unknown desc = failed to set up sandbox container "f55acf3c646821b942a51e020e6e8b4e5b9b4c022004d492d664087c1f3e44e8" network for pod "nginx-test-5d6675ddb4-9j6vg": NetworkPlugin cni failed to set up pod "nginx-test-5d6675ddb4-9j6vg_default" network: failed to set bridge addr: "cni0" already has an IP address different from 10.1.1.1/24
```

解决办法:重置网络
```
kubeadm reset
systemctl stop kubelet
systemctl stop docker
rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /etc/cni/
ifconfig cni0 down
ifconfig flannel.1 down
ifconfig docker0 down
ip link delete cni0
ip link delete flannel.1
systemctl start docker
```
