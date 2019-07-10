## 1.检查k8s集群状态
```
# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}
```

保证`ERROR`为空的

## 2.edgecontroller状态检查
```
# kubectl get deployment edgecontroller -n kubeedge
NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
edgecontroller   1         1         1            1           42d
```

保证检查最后一列的`AVAILABLE`至少为1

## 3.edge_core相关检查
- edge.yaml对应的配置是否已经修改,新添加的模块开关是否已经打开
- 需要检查进程是否存在
  ```
  ps -ef | grep edge
  ```
- ttu 对应的docker是否已经运行
  ```
  docker info
  ```

## 4.镜像仓库相关检查
- 检查harbor是否已经启动 `docker-compose ps`
- 检查镜像仓库是否能够访问
- 检查镜像仓库对应的镜像是否存在(名称+版本)
