# pod中的钩子
参考:
https://kubernetes.io/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/

2种钩子函数:
- PostStart：这个钩子在容器创建后立即执行。但是，并不能保证钩子将在容器`ENTRYPOINT`之前运行，因为没有参数传递给处理程序。主要用于资源部署、环境准备等。不过需要注意的是如果钩子花费太长时间以至于不能运行或者挂起， 容器将不能达到`running`状态。
- PreStop：这个钩子在容器终止之前立即被调用。它是阻塞的，意味着它是同步的，所以它必须在删除容器的调用发出之前完成。主要用于优雅关闭应用程序、通知其他系统等。如果钩子在执行期间挂起， Pod阶段将停留在`running`状态并且永不会达到failed状态。


使用例子：
这个例子在pod启动之前先输出一个语句到`/usr/share/message`；在关闭pod之前先优雅的干掉nginx
```
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/bin/sh","-c","nginx -s quit; while killall -0 nginx; do sleep 1; done"]
```

下面这个例子是因为退出的时候，容器销毁了，看不到preStop的效果，所以可以输出一个语句到主机上，作为参考：
```
apiVersion: v1
kind: Pod
metadata:
  name: hook-demo2
  labels:
    app: hook
spec:
  containers:
  - name: hook-demo2
    image: nginx
    ports:
    - name: webport
      containerPort: 80
    volumeMounts:
    - name: message
      mountPath: /usr/share/
    lifecycle:
      preStop:
        exec:
          command: ['/bin/sh', '-c', 'echo Hello from the preStop Handler > /usr/share/message']
  volumes:
  - name: message
    hostPath:
      path: /tmp
```


> 注意：另外钩子调用的日志没有暴露个给 Pod 的 `event`，所以只能通过`describe`来获取相关的信息，如果有错误将可以看到`FailedPostStartHook`或`FailedPreStopHook`这样的 `event`
