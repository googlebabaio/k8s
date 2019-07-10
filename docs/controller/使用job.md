直接上例子

```
apiVersion: batch/v1
kind: Job
metadata:
  name: job-demo
spec:
  template:
    metadata:
      name: job-demo
    spec:
      containers:
      - name: hello-world
        image: alpine
        command: ["echo", "Hello World!"]
      restartPolicy: Never
```
