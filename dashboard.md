<!-- toc -->

serviceaccount/admin-user created
serviceaccount/kubernetes-dashboard created

service/kubernetes-dashboard created

secret/kubernetes-dashboard-certs created

deployment.apps/kubernetes-dashboard created


role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created

clusterrolebinding.rbac.authorization.k8s.io/admin-user created
clusterrole.rbac.authorization.k8s.io/ui-admin created
clusterrole.rbac.authorization.k8s.io/ui-read created
rolebinding.rbac.authorization.k8s.io/ui-admin-binding created
rolebinding.rbac.authorization.k8s.io/ui-read-binding created



* * * * *
```
[root@master linux-amd64]# kubectl cluster-info
Kubernetes master is running at https://172.18.53.221:6443
CoreDNS is running at https://172.18.53.221:6443/api/v1/namespaces/kube-system/services/coredns:dns/proxy
kubernetes-dashboard is running at https://172.18.53.221:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
monitoring-grafana is running at https://172.18.53.221:6443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```


内网访问：
https://172.18.53.221:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy

外网访问：
https://39.108.71.10:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy


eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTVqMnhkIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI2Y2Q5N2FmNi1jMjFhLTExZTgtYmFiZi0wMDE2M2UwNDYzYTciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.Of_PCa0-hQepMTPqi34EzU-EIwKVOeGdd97tpuZO3SzWmJF8owBRTYz2qWqS0xP0gJMHiyj-9GbBgvlICEq1-Od2IHDyp302EesIDXRtgOuD1eouQ0yzxWBG0JYiilI67CIu_LspltC9HCJngCQo74ySH_LheZ2uNkE125IHbP10yysicPfTJuavg-Xpx3sWxfts7dQGSGOhqlkgPm3VvzuIS-Yrc-hXaY_3pw_qDdyXShHrmVikzloNpIWh3SDLezSiKKV_-BipTN64JYV2x9FQeKB-3wpDFxMOVtbYcIfELCQPrW9qnmEpWJDl84YCpbeum-7D5Lo0fIeXnr4-LA

![](images/screenshot_1538031299581.png)
