
1.为什么edgenode在容器启动失败后，在maskter logs pod的时候，会出现这个问题，
```
dial tcp 127.0.0.1:10250: getsockopt: connection refused
```


2.journalctl的使用
```
追踪日志
要主动追踪当前正在编写的日志，大家可以使用-f标记。同样功能类似为tail -f，只要不终止，会一直监控
journalctl -f

也许最有用的过滤方式是你感兴趣的单位。我们可以使用这个-u选项来过滤我们可以使用这个-u选项来过滤
journalctl -u

所以最终使用的命令是：
journalctl -f -u kubelet
```

3.节点的日志中出现这个报错
```
Turn up verbosity to see them
```


4.参考下这个文章
https://blog.csdn.net/qq_21816375/article/details/82803201

5.edgenode出现报错
```
Orphaned pod found - but volume paths are still present on disk
```

这个已经解决了,但是原因需要深入探索

k8s有个参考的地方:
https://github.com/kubernetes/kubernetes/issues/60987

6.docker 报错
```
/usr/bin/docker-current: Error response from daemon: oci runtime error: container_linux.go:247: starting container process caused "exec: \"-xx": executable file not found in $PATH".
```


```
standard_init_linux.go:207: exec user process caused "exec format error"
```

7.理解下hostNetwork网络
参考:
https://segmentfault.com/a/1190000016123122

8.树莓派的几个网页
https://www.cnblogs.com/zihunqingxin/p/8638514.html

9.k8s中limit或者request的总计,算法

10.记录一下centos中调时区和配置ntpd

11.研究下docker 2375,以及如何使用安全的方式访问
参考:
https://blog.csdn.net/ghostcloud2016/article/details/51539837

这个已经算解决了, 配置docker 启动的时候 -H TCP://0.0.0.0:2375 其他client就可以访问了,所以就反而需要把2375给限制点

12.HostAliases
https://ieevee.com/tech/2018/04/29/k8s-hostaliases.html
