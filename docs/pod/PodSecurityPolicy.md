<!-- toc -->

# PodSecurityPolicy是什么

PodSecurityPolicy是一种用来控制Pod安全相关配置的全局资源。

在开启RBAC的kubernetes集群上，如果允许用户使用kubectl，那么必须开启PodSecurityPolicy，否则用户可能会使用一些特权资源（例如privileged，hostNetwork，hostPath等等），影响node机器的稳定性。


参考:
https://kubernetes.io/docs/concepts/policy/pod-security-policy/#policy-reference
