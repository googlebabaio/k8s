

## 部署kube-stat-metrics

git clone --branch release-1.8 https://github.com/kubernetes/kube-state-metrics.git

## 使用operate来部署Prometheus


Create the monitoring stack using the config in the manifests directory:
```
# Create the namespace and CRDs, and then wait for them to be availble before creating the remaining resources
kubectl create -f manifests/setup
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
kubectl create -f manifests/
```

We create the namespace and CustomResourceDefinitions first to avoid race conditions when deploying the monitoring components. Alternatively, the resources in both folders can be applied with a single command `kubectl create -f manifests/setup -f manifests`, but it may be necessary to run the command multiple times for all components to be created successfullly.

And to teardown the stack:
```
kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup
```
