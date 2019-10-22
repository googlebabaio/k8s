## Prometheus
对于 K8S 的监控，我们选择 CNCF 旗下次于 K8S 毕业的项目 Prometheus 。

Prometheus 是一个非常灵活易于扩展的监控系统，它通过各种 exporter 暴露数据，并由 prometheus server 定时去拉数据，然后存储。

它自己提供了一个简单的前端界面，可在其中使用 PromQL 的语法进行查询，并进行图形化展示。

### 安装Prometheus
