## --restart-policy参数
带有参数 --restart-policy=Always 的资源将被部署为 Deployment
带有参数 --restart-policy=Never 的资源将被部署为 Pod。
同时 Kubectl 也会检查是否需要触发其他操作，例如：记录命令（用来进行回滚或审计）

## API 版本协商与 API 组
为了更容易地消除字段或者重新组织资源结构，Kubernetes 支持多个 API 版本，每个版本都在不同的 API 路径下，例如 /api/v1 或者 /apis/extensions/v1beta1。不同的 API 版本表明不同的稳定性和支持级别，更详细的描述可以参考 Kubernetes API 概述。

API 组主要作用是对类似资源进行分类，以便使得 Kubernetes API 更容易扩展。
API 的组别在 REST 路径或者序列化对象的 apiVersion 字段中指定。
例如：Deployment 的 API 组名是 apps，最新的 API 版本是 v1beta2，这就是为什么要在 Deployment manifests 顶部输入 apiVersion: apps/v1beta2。

为了提高性能，Kubectl 将 OpenAPI 模式缓存到了 ~/.kube/cache 目录。
如果想了解 API 发现的过程，可以尝试删除该目录并在运行 kubectl 命令时将 -v 参数的值设为最大值，然后将会看到所有试图找到这些 API 版本的 HTTP 请求。

## kubectl 通过以下顺序来找到 Kubeconfig
- 如果提供了 --kubeconfig 参数， Kubectl 就使用 --kubeconfig 参数提供的 Kubeconfig 文件。
- 如果没有提供 --kubeconfig 参数，但设置了环境变量 $KUBECONFIG，则使用该环境变量提供的 Kubeconfig 文件。
- 如果 --kubeconfig 参数和环境变量 $KUBECONFIG 都没有提供，Kubectl 就使用默认的 Kubeconfig 文件 $HOME/.kube/config。

## kubectl发出的HHP请求头中的内容
解析完 Kubeconfig 文件后，Kubectl 会确定当前要使用的上下文、当前指向的群集以及与当前用户关联的任何认证信息。如果用户提供了额外的参数（例如: --username），则优先使用这些参数覆盖 Kubeconfig 中指定的值。一旦拿到这些信息之后， Kubectl 就会把这些信息填充到将要发送的 HTTP 请求头中：
- x509 证书使用 tls.TLSConfig 发送（包括 CA 证书）
- bearer tokens 在 HTTP 请求头 Authorization 中发送
- 用户名和密码通过 HTTP 基本认证发送
- OpenID 认证过程是由用户事先手动处理的，产生一个像 Bearer Token 一样被发送的 Token

每次收到请求时，Apiserver 都会通过令牌链进行认证，直到某一个认证成功为止：
- x509 处理程序将验证 HTTP 请求是否是由 CA 根证书签名的 TLS 密钥进行编码的
- bearer token 处理程序将验证 --token-auth-file 参数提供的 Token 文件是否存在
- 基本认证处理程序确保 HTTP 请求的基本认证凭证与本地的状态匹配
