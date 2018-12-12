# 本文将介绍在k8s利用jenkins打造CI/CD
## devops
![](assets/markdown-img-paste-20181212121211490.png)

## CI/CD概念

![](assets/markdown-img-paste-20181212121059739.png)

- 持续集成（ContinuousIntegration，CI）：代码合并、构建、部署、测试都在一起，不断地执行这个过程，并对结果反馈。
- 持续部署（ContinuousDeployment，CD）：部署到测试环境、预生产环境、生产环境。
- 持续交付（ContinuousDelivery，CD）：将最终产品发布到生产环境，给用户使用。


## 传统的CI/CD流程
![](assets/markdown-img-paste-20181212130045540.png)

## 基于k8s的CI/CD流程
![](assets/markdown-img-paste-20181212121233146.png)

### 优点
 jenkins在整个工作中要做的事情

![](assets/markdown-img-paste-20181212122655231.png)

## 前置条件:
已经部署好了一套K8S
私有仓库
PV/PVC/SC

## 打造步骤:
- 部署jenkins-master
- 配置jenkins-master
- 制作jenkins-slave
- 创建pipeline项目
- 编写jenkinsfile文件
- 验证
- 优化改进
- 最终交付物
