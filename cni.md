CNI（Container Network Interface）是CNCF旗下的一个项目，由一组用于配置Linux容器的网络接口的规范和库组成，同时还包含了一些插件。CNI仅关心容器创建时的网络分配，和当容器被删除时释放网络资源。
通过此链接浏览该项目：https://github.com/containernetworking/cni

Kubernetes源码的`vendor/github.com/containernetworking/cni/libcni` 目录中已经包含了CNI的代码，也就是说kubernetes中已经内置了CNI

## 接口定义
CNI的接口中包括以下几个方法:
```
type CNI interface {
    AddNetworkList(net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)
    DelNetworkList(net *NetworkConfigList, rt *RuntimeConf) error

    AddNetwork(net *NetworkConfig, rt *RuntimeConf) (types.Result, error)
    DelNetwork(net *NetworkConfig, rt *RuntimeConf) error
}
```


> 参考
https://github.com/containernetworking/cni
https://github.com/containernetworking/plugins
[Container Networking Interface Specification](https://github.com/containernetworking/cni/blob/master/SPEC.md#container-networking-interface-specification)
[CNI Extension conventions](https://github.com/containernetworking/cni/blob/master/CONVENTIONS.md)
[Kubernetes网络插件CNI学习整理](https://blog.csdn.net/u010129347/article/details/78800065)
[CNCF CNI系列之一：浅谈kubernetes的网络与CNI(以flannel为例)](https://blog.csdn.net/cloudvtech/article/details/79753123)
[容器CNI完全解读（一）](https://blog.csdn.net/u010278923/article/details/75229024)
[kubernetes系列之十三：POD多网卡方案multus-cni之通过CRD给POD配置自由组合网络](https://blog.csdn.net/cloudvtech/article/details/80238843)
[Kubernetes的网络接口CNI及灵雀云的实践](https://blog.csdn.net/alauda_andy/article/details/80132922)
[借助 Calico，管窥 Kubernetes 网络策略](https://blog.csdn.net/qq_34463875/article/details/62928676)
[k8s技术预研12--kubernetes的常见开源网络组件](https://blog.csdn.net/watermelonbig/article/details/80720378)
[Kubernetes-基于flannel的集群网络](https://www.kubernetes.org.cn/4105.html)