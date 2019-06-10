# k8s中的 LimitRange

LimitRange(简称limits)基于namespace的资源管理，包括pod和container的最小、最大和default、defaultrequests等。

一旦创建limits，以后创建资源时，K8S将该limits资源限制条件默认/强制给pod，创建后发现不符合规则，将暂停创建pod。
在创建资源时，用户可以为pod自定义资源管理限制，在创建时会去检查和匹配limits值，发现不匹配将在创建时报错。创建后，该pod的资源使用遵守自定义规则，而不会遵守namespace的limits限制。

## 例子
```
apiVersion: v1
kind: LimitRange
metadata:
  namespace: default
  name: qmxyfzx-limitrange
  labels:
    project: qmxyfzx
    app: limitrange
    version: v1
spec:
  limits:
  - max:
      cpu: 1
      memory: 1Gi
    min:
      cpu: 0.05
      memory: 64Mi
    type: Pod
    #注意pod只能这么多参数
  - default:
      cpu: 0.2
      memory: 200Mi
    defaultRequest:
      cpu: 0.01
      memory: 16Mi
    max:
      cpu: 0.25
      memory: 256Mi
    min:
      cpu: 0.005
      memory: 8Mi
    #container只能这么多参数
    type: Container
```

## 参数说明
### pod可以有的参数
- max：表示pod中所有容器资源的Limit值和的上限，也就是整个pod资源的最大Limit，如果pod定义中的Limit值大于LimitRange中的值，则pod无法成功创建。
- min：表示pod中所有容器资源请求总和的下限，也就是所有容器request的资源总和不能小于min中的值，否则pod无法成功创建。
- maxLimitRequestRatio：表示pod中所有容器资源请求的Limit值和request值比值的上限，例如该pod中cpu的Limit值为3，而request为0.5，此时比值为6，创建pod将会失败。

### container可以有的参数
- 在container的部分，max、min和maxLimitRequestRatio的含义和pod中的类似，只不过是针对单个的容器而言。下面说明几个情况：
```
如果container设置了max， pod中的容器必须设置limit，如果未设置，则使用defaultlimt的值，如果defaultlimit也没有设置，则无法成功创建
如果设置了container的min，创建容器的时候必须设置request的值，如果没有设置，则使用defaultrequest，如果没有defaultrequest，则默认等于容器的limit值，如果limit也没有，启动就会报错
```
- defaultlimit ：默认时，限制使用的资源是：CPU 0.2个核，内存200M
- defaultRequest：默认时，最低保证可以使用的资源是： CPU 0.01个核，内存16M




## 使用规则

- 1、每个namespace应该只有一个limits；

- 2、limits值设置：
```
每容器（type： container）
    max>=default>=defaultRequest>min
每pod（type： pod）
    max>=min
整个
    容器的max*容器数<=pod的max
    容器的min*容器数<=pod的min
```
- 3、创建资源时，pod自定义资源限制的规则：
```
自定义的单个request>=limits的容器的defaultrequets
自定义的request的总和>=limits的pod的min
自定义的单个limit<=limits的容器的requets
自定义的limit的总和<=limits的pod的max
```

## 使用建议
- 只使用limits的pod或者container中的一种，尽量不使用同时使用，特别在pod中有多容器需求的情况下。
- 尽量使用max，尽量不同时使用max和min
- 由于limits会针对该namespace下的所有新建的pods，所以在该namespace下应该运行哪些资源需求相同的业务
- 在复杂的limits配置下，不要在创建资源时使用自定义配置。
