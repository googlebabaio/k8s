k8s允许我们将不同类型的volume挂载到容器的特定目录下。例如，我们可以将configmap的数据以volume的形式挂到容器下。


定义一个configmap，其中的数据以 key:value 的格式体现。

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.level: very
  special.type: |-
    property.1=value-1
    property.2=value-2
    property.3=value-3
```

创建一个Pod，其挂载上面定义的cm，并在启动时查看挂载目录 /etc/config/ 下有哪些文件。

```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        # Provide the name of the ConfigMap containing the files you want
        # to add to the container
        name: special-config
  restartPolicy: Never
```

这里有几点需要注意:

- 挂载目录下的文件名称，即为cm定义里的key值。
- 挂载目录下的文件的内容，即为cm定义里的value值。value可以多行定义，这在一些稍微复杂的场景下特别有用，比如 my.cnf。
- 如果挂载目录下原来有文件，挂载后将不可见（AUFS）。


有的时候，我们希望将文件挂载到某个目录，但希望只是挂载该文件，不要影响挂载目录下的其他文件。

这个时候就可以用subPath: `Path within the volume from which the container's volume should be mounted`。
subPath 的目的是为了在单一Pod中多次使用同一个volume而设计的。

例如:
像下面的LAMP，可以将同一个volume下的 mysql 和 html目录，挂载到不同的挂载点上，这样就不需要为 mysql 和 html 单独创建volume了。
```
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd"
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```


```
containers:
- volumeMounts:
  - name: demo-config
    mountPath: /etc/special.type
    subPath: special.type
volumes:
- name: demo-config
  configMap:
    name: special-config
```

# 参考
https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath
