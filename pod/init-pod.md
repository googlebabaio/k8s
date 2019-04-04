

Init Container就是用来做初始化工作的容器，可以是一个或者多个，如果有多个的话，这些容器会按定义的顺序依次执行，只有所有的Init Container执行完后，主容器才会被启动。我们知道一个Pod里面的所有容器是共享数据卷和网络命名空间的，所以Init Container里面产生的数据可以被主容器使用到的。

![](assets/markdown-img-paste-20190404173813723.png)



init container / main container 的生命周期

从上面这张图可以直观的看到PostStart和PreStop包括liveness和readiness是属于主容器的生命周期范围内的，而Init Container是独立于主容器之外的，当然它们都属于Pod的生命周期范畴之内的。
