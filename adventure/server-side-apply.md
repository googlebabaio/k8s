Server-side Apply

Kubernetes 是一个声明式的资源管理系统。用户在本地定义期望的状态，然后通过 kubectl apply 去跟更新当前集群状态中被用户指定的那一部分。然而它远没有听起来那么简单...

原来的 kubectl apply 是基于客户端实现的。Apply 的时候不能简单地替换掉单个资源的整体状态，因为还有其他人也会去更改资源，比如 controllers、admissions、webhooks。那么怎样保证在改一个资源的同时，不会覆盖掉别人的改动呢？

于是就有了现有的 3 way merge：用户把 last applied state 存在 Pod annotations 里，在下次 apply 的时候根据 (最新状态，last applied，用户指定状态) 做 3 way diff，然后生成 patch 发送到 APIServer。但是这样做还是有问题！Apply 的初衷是让个体去指定哪些资源字段归他管理。

但是原有实现既不能阻止不同个体之间互相篡改字段，也没有在冲突发生时告知用户和解决。举个例子，笔者原来在 CoreOS 工作时，产品里自带的 controller 和用户都会去更改 Node 对象的一些特殊 labels，结果出现冲突，导致集群出故障了只能派人去修。

这种克苏鲁式的恐惧笼罩着每一个 k8s 用户，而现在我们终于迎来了胜利的曙光——那就是服务器端 apply。APIServer 会做 diff 和 merge 操作，很多原本易碎的现象都得到了解决。更重要的是，相比于原来用 last-applied annotations，服务器端 apply 新提供了一种声明式 API (叫 ManagedFields) 来明确指定谁管理哪些资源字段。而当发生冲突时，比如 kubectl 和 controller 都改同一个字段时，非 Admin（管理员）的请求会返回错误并且提示去解决。
