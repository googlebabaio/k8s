## 概述
K8S 中有一套默认的集群内 DNS 服务，我们通常把它叫做 kube-dns，它基于 SkyDNS，为我们在服务注册发现方面提供了很大的便利。

- 在 K8S 1.9 版本中加入并进入 Alpha 阶段
- CoreDNS 在 K8S 1.13 版本中才正式成为默认的 DNS 服务



## CoreDNS 是什么

- CoreDNS 是一个独立项目，它不仅可支持在 K8S 中使用，你也可以在你任何需要 DNS 服务的时候使用它。
- CoreDNS 使用 Go 语言实现，部署非常方便。
- 它的扩展性很强，很多功能特性都是通过插件完成的，它不仅有大量的内置插件，同时也有很丰富的第三方插件。甚至你自己写一个插件也非常的容易。

## 安装使用 CoreDNS
地址: https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns

```
[root@k8s-master ~]# kubectl get deploy,pod -n kube-system
NAME                                         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/coredns                2         2         2            2           76d

NAME                                       READY     STATUS    RESTARTS   AGE
pod/coredns-5f94b495b5-6k9vr               1/1       Running   7          70d
pod/coredns-5f94b495b5-px7p8               1/1       Running   1          47d
```

## 验证 CoreDNS 功能
```
#  kubectl -n kube-system get configmap coredns -o yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local. in-addr.arpa ip6.arpa {
            pods insecure
            upstream
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        proxy . /etc/resolv.conf
        cache 30
    }
kind: ConfigMap
metadata:
  creationTimestamp: 2018-10-09T08:51:46Z
  labels:
    addonmanager.kubernetes.io/mode: EnsureExists
  name: coredns
  namespace: kube-system
  resourceVersion: "14409"
  selfLink: /api/v1/namespaces/kube-system/configmaps/coredns
  uid: 8da9c976-cba0-11e8-a304-525400f9dfac
```

Corefile 是它的配置文件，可以看到它启动了类似 `kubernetes`, `prometheus` 等插件。

注意 kubernetes 插件的配置，使用的域是 `cluster.local.` ，这也是上面我们提到域名格式时候后半部分未解释的部分。

至于 prometheus 插件，则是监听在 9153 端口上提供了符合 Prometheus 标准的 metrics 接口，可用于监控等。
