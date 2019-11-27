

git clone https://github.com/coreos/kube-prometheus.git


```
[root@host-192-168-3-15 manifests]# kubectl get all
NAME                                       READY   STATUS              RESTARTS   AGE
pod/alertmanager-main-0                    0/2     ContainerCreating   0          47s
pod/alertmanager-main-1                    0/2     ContainerCreating   0          47s
pod/alertmanager-main-2                    0/2     ContainerCreating   0          47s
pod/grafana-5db74b88f4-lh5fm               0/1     ContainerCreating   0          46s
pod/kube-state-metrics-54f98c4687-ns7wd    0/3     ContainerCreating   0          46s
pod/node-exporter-9f94p                    0/2     ContainerCreating   0          46s
pod/node-exporter-jlr2s                    0/2     ContainerCreating   0          46s
pod/node-exporter-vltft                    0/2     ContainerCreating   0          46s
pod/prometheus-adapter-8667948d79-98jvn    0/1     ContainerCreating   0          46s
pod/prometheus-k8s-0                       0/3     ContainerCreating   0          47s
pod/prometheus-k8s-1                       0/3     ContainerCreating   0          47s
pod/prometheus-operator-5f759fd859-c65k4   1/1     Running             0          5m50s

NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-main       ClusterIP   10.111.148.199   <none>        9093/TCP                     48s
service/alertmanager-operated   ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   48s
service/grafana                 ClusterIP   10.98.230.84     <none>        3000/TCP                     47s
service/kube-state-metrics      ClusterIP   None             <none>        8443/TCP,9443/TCP            47s
service/node-exporter           ClusterIP   None             <none>        9100/TCP                     47s
service/prometheus-adapter      ClusterIP   10.98.159.130    <none>        443/TCP                      47s
service/prometheus-k8s          ClusterIP   10.109.213.110   <none>        9090/TCP                     47s
service/prometheus-operated     ClusterIP   None             <none>        9090/TCP                     47s
service/prometheus-operator     ClusterIP   None             <none>        8080/TCP                     5m51s

NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/node-exporter   3         3         0       3            0           kubernetes.io/os=linux   47s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana               0/1     1            0           47s
deployment.apps/kube-state-metrics    0/1     1            0           47s
deployment.apps/prometheus-adapter    0/1     1            0           47s
deployment.apps/prometheus-operator   1/1     1            1           5m51s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-5db74b88f4               1         1         0       47s
replicaset.apps/kube-state-metrics-54f98c4687    1         1         0       47s
replicaset.apps/prometheus-adapter-8667948d79    1         1         0       47s
replicaset.apps/prometheus-operator-5f759fd859   1         1         1       5m51s

NAME                                 READY   AGE
statefulset.apps/alertmanager-main   0/3     48s
statefulset.apps/prometheus-k8s      0/2     47s
```
