<!--toc -->
# Secret
>  参考 [官方文档之Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

##  secret概览
secret是k8s里的一个对象， 它用于存储一些敏感数据，比如说密码， token， 密钥等等。这类信息如果直接明文放在镜像或者pod里， 比较不安全。 用secret来保存会更加安全，可以防止意外泄露。
secret如何被使用？
> 有两种使用方式： 1.  作为一个卷被pod挂载  2. kubelet 为pod拉取镜像的时候使用

内置的secret
>   Service Account创建时会自动secret，供集群访问API时使用

使用kubectl命令创建secret
1. 先在本地创建两个文本用于存放username和password
```shell
# Create files needed for rest of example.
$ echo -n "admin" > ./username.txt
$ echo -n "1f2d1e2e67df" > ./password.txt
```
2. 用 kubectl命令创建Secret，把这两个文件的内容打包进Secret， 并在apiserver中创建API 对象
```shell
$ kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
secret "db-user-pass" created
```
3. 查看secret
```shell
$ kubectl get secrets
NAME                  TYPE                                  DATA      AGE
db-user-pass          Opaque                                2         51s

$ kubectl describe secrets/db-user-pass
Name:            db-user-pass
Namespace:       default
Labels:          <none>
Annotations:     <none>

Type:            Opaque

Data
====
password.txt:    12 bytes
username.txt:    5 bytes
```
4. 解码secret的加密内容
```shell
$ kubectl get secret mysecret -o yaml
apiVersion: v1
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
kind: Secret
metadata:
  creationTimestamp: 2016-01-22T18:41:56Z
  name: mysecret
  namespace: default
  resourceVersion: "164619"
  selfLink: /api/v1/namespaces/default/secrets/mysecret
  uid: cfee02d6-c137-11e5-8d73-42010af00002
type: Opaque
$ echo "MWYyZDFlMmU2N2Rm" | base64 --decode
1f2d1e2e67df
```

通过yaml编排文件来创建secret
```shell
# 先把内容用base64转换一下
$ echo -n "admin" | base64
YWRtaW4=
$ echo -n "1f2d1e2e67df" | base64
MWYyZDFlMmU2N2Rm
```
然后创建如下yaml文件
```yml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```
用kubectl创建secret
```shell
$ kubectl create -f ./secret.yaml
secret "mysecret" created
```

在pod中使用secret
```yml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```
username.txt和password.txt会出现在/etc/foo/username.txt, /etc/foo/password.txt路径下
你也可以为username指定不同的路径，用法如下(spec.volumes[].secret.items)
```yml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
```
此时， username的值会出现在/etc/foo/my-group/my-username下，而不是/etc/foo/username路径
为secret文件分配读写权限(spec.volumes[].secret.defaultMode)
```yml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      defaultMode: 256

```
注意这里的256，转成8进制就是0400， JSON不支持8进制，所以在指定值得时候要转换成10进制
如何在pod里读取挂载的secret的内容,  secret的内容挂载到pod里之后变成了文件，直接在挂载的对应路径下读取即可
```shell
$ ls /etc/foo/
username
password
$ cat /etc/foo/username
admin
$ cat /etc/foo/password
1f2d1e2e67df
```

挂载到pod里的secret的内容， 会进行自动更新， 如果你更新了secret对象的内容， pod里对应的内容也会更新。 内容更新的时间为kubelet 的sync周期 + secret cache的ttl时间。

将Secret作为环境变量使用(env[x].valueFrom.secretKeyRef)
```yml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
  restartPolicy: Never
```
从环境变量中获取Secret的值
```shell
$ echo $SECRET_USERNAME
admin
$ echo $SECRET_PASSWORD
1f2d1e2e67df
```

使用 imagePullSecrets
>  imagePullSecret是一个secret， 它会把docker registry的密码传递给kubelet，让kubelet代表pod下载镜像
> 你可以从创建一个imagePullSecrets， 并在serviceaccount中指向这个imagePullSecrets，所有使用这个serviceaccount的pod， 会自动设置它的imagePullSecret 字段

一些限制
1. secret需要先于使用它的pod创建， 否则pod的启动不会成功
2. secret只能被相同namespace的pod使用
3. 单个secret的大小最大为1MB， 这个限制的出发点是为了避免过大的secret的会耗尽apiserver和kubelet的内存， 因此不建议创建非常大的secret。 然而创建大量小的secret对象仍然会耗尽内存。 这个目前无法对其限制。 对secret使用内存的复杂限制是规划中的一个特性。
4. kubelet目前只支持从apiserver创建的pod使用secret， 包括kubectl直接创建或者从复制控制器中的非直接创建方式。 其它方式，如使用 kubelet的 --manifest-url或 --config参数创建的pod，不能使用secret。
5. secretKeyRef指定的key如果不存在， 那么pod的启动不会成功

secret和pod的生命周期的交互
1. pod通过api创建时， 这时不会检查引用的secret是否存在
2. 当pod被调度时，宿主机上的kubelet就会检查secret的值
3. 当secret不能被获取时， kubelet会做周期性的尝试， 并会记录一个event说明为什么没有启动
4. 当secret被获取时， kubelet就会创建和挂载一个包含这个secret的存储卷
5. 在所有的卷被创建和挂载之前， pod不会启动
