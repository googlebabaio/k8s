# 认证的几种方式
![](../images/screenshot_1538118926479.png)
三种方式：
* 基本basic
* 令牌token
* 双向tls

# kubectl发出命令请求的流程
## kubectl 通过以下顺序来找到 Kubeconfig 文件：

- 如果提供了 --kubeconfig 参数， Kubectl 就使用 --kubeconfig 参数提供的 Kubeconfig 文件
- 如果没有提供 --kubeconfig 参数，但设置了环境变量 $KUBECONFIG，则使用该环境变量提供的 Kubeconfig 文件
- 如果 --kubeconfig 参数和环境变量 $KUBECONFIG 都没有提供，Kubectl 就使用默认的 Kubeconfig 文件 $HOME/.kube/config


每次收到请求时，Apiserver 都会通过令牌链进行认证，直到某一个认证成功为止：
- x509 处理程序将验证 HTTP 请求是否是由 CA 根证书签名的 TLS 密钥进行编码的
- bearer token 处理程序将验证 --token-auth-file 参数提供的 Token 文件是否存在
- 基本认证处理程序确保 HTTP 请求的基本认证凭证与本地的状态匹配

解析完 Kubeconfig 文件后，Kubectl 会确定当前要使用的上下文、当前指向的群集以及与当前用户关联的任何认证信息。如果用户提供了额外的参数（例如: --username），则优先使用这些参数覆盖 Kubeconfig 中指定的值。一旦拿到这些信息之后， Kubectl 就会把这些信息填充到将要发送的 HTTP 请求头中：
- x509 证书使用 tls.TLSConfig 发送（包括 CA 证书）
- bearer tokens 在 HTTP 请求头 Authorization 中发送
- 用户名和密码通过 HTTP 基本认证发送
- OpenID 认证过程是由用户事先手动处理的，产生一个像 Bearer Token 一样被发送的 Token

## 认证
现在kubectl的请求已经成功发送了，接下来轮到 Kube-Apiserver 闪亮登场。

### 认证的定义
Kube-Apiserver 是客户端和系统组件用来保存和检索集群状态的主要接口。为了执行相应的功能，Kube-Apiserver 需要能够验证请求者是合法的，这个过程被称为认证。

### 认证的过程
当 Kube-Apiserver 第一次启动时，它会查看用户提供的所有 CLI 参数，并组合成一个合适的令牌列表。
如果提供了 --client-ca-file 参数，则会将 x509 客户端证书认证添加到令牌列表中；如果提供了 --token-auth-file 参数，则会将 breaer token 添加到令牌列表中。

每次收到请求时，Apiserver 都会通过令牌链进行认证，直到某一个认证成功为止：
- x509 处理程序将验证 HTTP 请求是否是由 CA 根证书签名的 TLS 密钥进行编码的
- bearer token 处理程序将验证 --token-auth-file 参数提供的 Token 文件是否存在
- 基本认证处理程序确保 HTTP 请求的基本认证凭证与本地的状态匹配
