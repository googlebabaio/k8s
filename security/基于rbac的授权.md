参考：
[Kubernetes中的角色访问控制机制（RBAC）支持](https://www.kubernetes.org.cn/1879.html)
[Kubernetes-基于RBAC的授权](https://www.kubernetes.org.cn/4062.html)
[kubernetes RBAC认证简介](https://blog.csdn.net/qq_21816375/article/details/80039345)

## 概述 
RBAC是Role-Based Access Control的简称，中文为基于角色的访问控制 
RBAC使用“rbac.authorization.k8s.io”API组来驱动授权决策，允许管理员通过Kubernetes API动态配置策略。

从1.8开始，RBAC模式处于稳定版本，并由`rbac.authorization.k8s.io/v1` API提供支持。

要启用RBAC，请使用`--authorization-mode=RBAC`启动apiserver。

