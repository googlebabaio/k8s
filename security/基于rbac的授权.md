<!-- toc -->

# RBAC概述
RBAC是Role-Based Access Control的简称，中文为基于角色的访问控制
RBAC使用“rbac.authorization.k8s.io”API组来驱动授权决策，允许管理员通过Kubernetes API动态配置策略。

从1.8开始，RBAC模式处于稳定版本，并由`rbac.authorization.k8s.io/v1` API提供支持。

要启用RBAC，请使用`--authorization-mode=RBAC`启动apiserver。

## Role and ClusterRole
- Role：在namespace内
- ClusterRole：整集群内

如下Role在 namespace default里具有pods的读权限。
```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

ClusterRole除了具备和Role一样可以配置的权限，还有如下几种：
- cluster-scoped resources (like nodes)
- non-resource endpoints (like “/healthz”)
- namespaced resources (like pods) across all namespaces (needed to run kubectl get pods –all-namespaces, for example)

如下ClusterRole具备在集群范围内的所有secret的读权限。
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

## RoleBinding and ClusterRoleBinding
顾名思义，RoleBinding就是在某一namespace范围内，将Role授予某一个用户或一组用户（(users, groups, or service accounts)。
相对的，ClusterRoleBinding是在集群范围内授权的。

如下在namespace default里，将Role pod-reader授予给用户sure
```
# This role binding allows "sure" to read pods in the "default" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: sure # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
RoleBinding也可以将集群范围的ClusterRole授予给某namespace里的用户，这样管理员可以配置整集群的ClusterRole，然后在多个namespace里复用。

例如，下面在namespace development里，将ClusterRole secret-read授予给用户dandan，这样dandan只能在namespace development里读取secret，对于其他namespace里的secret，仍然是没有权限的。
```
# This role binding allows "dandan" to read secrets in the "development" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-secrets
  namespace: development # This only grants permissions within the "development" namespace.
subjects:
- kind: User
  name: dandan # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

ClusterRoleBinding 则是在整个集群范围内给用户授予权限。下面配置将在集群范围内，允许manager组具有所有secret的读权限。
```
# This cluster role binding allows anyone in the "rzpadmin" group to read secrets in any namespace.
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: rzpadmin # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

# 参考：
[Kubernetes中的角色访问控制机制（RBAC）支持](https://www.kubernetes.org.cn/1879.html)
[Kubernetes-基于RBAC的授权](https://www.kubernetes.org.cn/4062.html)
[kubernetes RBAC认证简介](https://blog.csdn.net/qq_21816375/article/details/80039345)
