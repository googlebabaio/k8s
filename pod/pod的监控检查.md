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
