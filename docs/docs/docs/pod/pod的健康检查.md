<!-- toc -->
参考:
https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/


## container中有2种探针
- liveness:存活探针
- readiness:可读性探针

### 存活探针 - liveness
* kubelet 通过使用 liveness probe 来确定你的应用程序是否正在运行，通俗点将就是是否还活着。一般来说，如果你的程序一旦崩溃了， Kubernetes 就会立刻知道这个程序已经终止了，然后就会重启这个程序。而我们的 liveness probe 的目的就是来捕获到当前应用程序还没有终止，还没有崩溃，如果出现了这些情况，那么就重启处于该状态下的容器，使应用程序在存在 bug 的情况下依然能够继续运行下去。

### 可读性探针 - readiness
* kubelet 使用 readiness probe 来确定容器是否已经就绪可以接收流量过来了。这个探针通俗点讲就是说是否准备好了，现在可以开始工作了。只有当 Pod 中的容器都处于就绪状态的时候 kubelet 才会认定该 Pod 处于就绪状态，因为一个 Pod 下面可能会有多个容器。当然 Pod 如果处于非就绪状态，那么我们就会将他从我们的工作队列(实际上就是我们后面需要重点学习的 Service)中移除出来，这样我们的流量就不会被路由到这个 Pod 里面来了。


## 探针的使用方式
- exec：执行一段命令
- http：检测某个 http 请求
- tcpSocket：使用此配置， kubelet 将尝试在指定端口上打开容器的套接字。如果可以建立连接，容器被认为是健康的，如果不能就认为是失败的。实际上就是检查端口

## 参数说明
- periodSeconds: kubelet需要每隔多少秒执行一次xxx probe
- initialDelaySeconds : 指定kubelet在该执行第一次探测之前需要等待的秒数
- timeoutSeconds：探测超时时间，默认1秒，最小1秒。
- successThreshold：探测失败后，最少连续探测成功多少次才被认定为成功。默认是 1，但是如果是`liveness`则必须是 1。最小值是 1。
- failureThreshold：探测成功后，最少连续探测失败多少次才被认定为失败。默认是 3，最小值是 1。
-
## 示例

### 存活探针示例
#### 使用执行命令的方式
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

#### 使用http请求的方式
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/liveness
    args:
    - /server
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```

在` k8s.gcr.io/liveness`这个镜像中部署了一个go的程序,
```
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    duration := time.Now().Sub(started)
    if duration.Seconds() > 10 {
        w.WriteHeader(500)
        w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
    } else {
        w.WriteHeader(200)
        w.Write([]byte("ok"))
    }
})
```
在这个例子中,container启动10秒内,执行了`/healthz` 这个命令后,都是返回的200,对于程序来说,我们认为200-400之间的值都是正常的,而当超过10s后,就返回500了,这个时候存活探针就认为不正常了.

#### 使用tcp socket的方式
```
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: k8s.gcr.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

TCP 检查的配置与 HTTP 检查非常相似，只是将httpGet替换成了tcpSocket。 该探针会去连接容器的8080端，如果连接成功，则该 Pod 将被标记为就绪状态。然后Kubelet将每隔10秒钟执行一次该检查。

### 可读性探针示例
从上面的YAML文件可以看出readiness probe的配置跟liveness probe很像，基本上一致的。唯一的不同是使用readinessProbe而不是livenessProbe。两者如果同时使用的话就可以确保流量不会到达还未准备好的容器，准备好过后，如果应用程序出现了错误，则会重新启动容器。


### 实际的一个treafik的探针示例
test的deployment
```
apiVersion: v1
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: test
  namespace: default
  labels:
    test: alpine
spec:
  replicas: 1
  selector:
    matchLabels:
      test: alpine
  template:
    metadata:
      labels:
        test: alpine
        name: test
    spec:
      containers:
      - image: mritd/alpine:3.4
        name: alpine
        resources:
          limits:
            cpu: 200m
            memory: 30Mi
          requests:
            cpu: 100m
            memory: 20Mi
        ports:
        - name: http
          containerPort: 80
        args:
        command:
        - "bash"
        - "-c"
        - "echo ok > /tmp/health;sleep 120;rm -f /tmp/health"
        livenessProbe:
          exec:
            command:
            - cat
            - /tmp/health
          initialDelaySeconds: 20
```

test的svc
```
apiVersion: v1
kind: Service
metadata:
  name: test
  labels:
    name: test
spec:
  ports:
  - port: 8123
    targetPort: 80
  selector:
    name: test
```

test的ingress
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
spec:
  rules:
  - host: test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: test
          servicePort: 8123
```


全部创建好以后，进入 traefik ui 界面，可以观察到每隔 2 分钟健康检查失败后，kubernetes 重建 pod，同时 traefik 会从后端列表中移除这个 pod

其他更多请参考官方文档 https://docs.traefik.io/
