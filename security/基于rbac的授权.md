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

## 涉及到的资源
大部分资源是直接用Restful API URL的资源字符串来表示的，比如 pods。但某些k8s API还由子资源（subresource），例如容器的日志：
```
GET /api/v1/namespaces/{namespace}/pods/{name}/log
```

pods是一级资源（注意k8s没有把namespace作为资源），用pods表示；log是二级资源，用`pods/log`来表示。如下将创建一个允许读取pods和pods下日志的Role。
```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-and-pod-logs-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]
```

也可以通过 resourceName 指定单个资源（而不是一类资源）。如下Role指定的是对 my-configmap 这个cm的在namespace default里的 get和update 权限。
```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: configmap-updater
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-configmap"]
  verbs: ["update", "get"]
```
注意，指定了单个资源后，verbs里就不能由 `list, watch, create, or deletecollection` 了，因为这几个verbs对应的API URL里不会出现resource name。

## 聚合clusterole
- k8s 1.9版本以后，可以用 `aggregationRule` 来聚合其他`ClusterRole`，从而创建一个新的具有更多权限的ClusterRole。
- 聚合的方法是通过matchLabels（即`rbac.example.com/aggregate-to-monitoring: "true"``），来匹配所有metadata符合该label的ClusterRole。
- aggregationRule不需要配置 rules 段，它是由controller收集所有匹配的ClusterRole的rules后填充的。

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: [] # Rules are automatically filled in by the controller manager.
```
注意，创建新的符合matchLabel的clusterRole，controller会将新的rules添加到aggregationRule。如下会将 monitoring-endpoints的rules添加到上面的ClusterRole monitoring。
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
# These rules will be added to the "monitoring" role.
rules:
- apiGroups: [""]
  Resources: ["services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
```

默认的面向用户的role使用了aggregationRule。这样admin可以自动拥有 `CustomResourceDefinitions` CRD的权限。


### 集群范围内的 ClusterRole

- admin会聚合所有label为`rbac.authorization.k8s.io/aggregate-to-admin: "true"`的ClusterRole
- 而 view 则会聚合所有label为`rbac.authorization.k8s.io/aggregate-to-view: "true"`的ClusterRole。


```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-admin: "true"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: view
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-view: "true"
```
配合下面的clusterrole使用
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aggregate-cron-tabs-edit
  labels:
    # Add these permissions to the "admin" and "edit" default roles.
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
rules:
- apiGroups: ["stable.example.com"]
  resources: ["crontabs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aggregate-cron-tabs-view
  labels:
    # Add these permissions to the "view" default role.
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
- apiGroups: ["stable.example.com"]
  resources: ["crontabs"]
  verbs: ["get", "list", "watch"]
```

参考:
https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-examples

## 涉及到的Subjects
RoleBinding、ClusterRoleBinding将Role、ClusterRole绑定到 `subject`。subject可以是** user，group，serviceAccount**。

- user：k8s并不管理user，而是authenticator（例如我们用的是dex）在管理。user可以是字符串（如”jane”），email（如bob@example.com），ID。这取决于管理员配置认证的时候，--oidc-username-claim配置的是什么。我们这里用的是name。system:开头的用户是保留给k8s用的。

- group：group是由 Authenticator modules 在管理。可以是像user一样的普通group，也可以是 system:开头的k8s的组。

- serviceaccount：由k8s管理。sa区别于user，其代表的是service，用来供service之间访问用，例如service A调用 service B的API，那么可以为service A创建一个 sa ，然后赋予该sa访问 service B API的权限。sa类似appid的概念。sa可以用 kubectl get sa来查看。


注意group的两个例子:
```
subjects:
- kind: Group
  name: "frontend-admins"
  apiGroup: rbac.authorization.k8s.io
---
# namespace qa下的sa
subjects:
- kind: Group
  name: system:serviceaccounts:qa
  apiGroup: rbac.authorization.k8s.io
---
# 所有sa
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
---
# 所有已认证用户
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
---
# 所有未认证用户
subjects:
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
# 所有用户
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
```
## 默认的 Roles 和 Role Bindings
API Server内置了若干clusterRole和clusterRoleBinding，一般是以system:开头，label为`kubernetes.io/bootstrapping=rbac-defaults`。尽量不要去修改他们，否则会引起系统级别的问题，因为可能某些服务会因为权限不足而无法正常运行。

### Auto-reconciliation
万一不小心把上面的clusterRole或者clusterRoleBinding给改错了呢？API Master具备 `Auto-reconciliation`的功能，即，API Master每次重启的时候，都会重新恢复默认的clusterRole/ClusterRoleBiding。

如果确实要改，可以修改其`rbac.authorization.kubernetes.io/autoupdate`为false，这样API Server就不会恢复其默认值了。

### user-facing roles
API Server还内置了一些不以system:开头的clusterRole:

- cluster-admin：集群超级管理员。resources、verbs匹配全是 *
- admin/edit/view：在某namespace中授权

## 使用kubectl创建roleBiding/clusterRoleBinding
创建某一namespace下的roleBinding:
```
kubectl create rolebinding bob-admin-binding --clusterrole=admin --user=bob --namespace=acme
kubectl create rolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp --namespace=acme
```

创建clusterRoleBiding:
```
kubectl create clusterrolebinding root-cluster-admin-binding --clusterrole=cluster-admin --user=root
kubectl create clusterrolebinding kubelet-node-binding --clusterrole=system:node --user=kubelet
kubectl create clusterrolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp
```

## Service Account Permissions
默认RBAC不会给 kube-system 之外的sa授予权限。

>https://kubernetes.io/docs/reference/access-authn-authz/rbac/#service-account-permissions
详细描述了权限管理从严格到宽松的各种sa授权方式。总之，管理越严格，管理员越忙。

# 参考：
[Kubernetes中的角色访问控制机制（RBAC）支持](https://www.kubernetes.org.cn/1879.html)
[Kubernetes-基于RBAC的授权](https://www.kubernetes.org.cn/4062.html)
[kubernetes RBAC认证简介](https://blog.csdn.net/qq_21816375/article/details/80039345)
https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-examples
https://kubernetes.io/docs/reference/access-authn-authz/rbac/#discovery-roles
