
直接上例子
```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cronjob-demo
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello-word
            image: alpine
            command: ["echo", "CronJob"]
          restartPolicy: OnFailure
```
