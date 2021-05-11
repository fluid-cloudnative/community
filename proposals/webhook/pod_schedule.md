# Pod Schedule Design Doc
# Motivation
用户在使用了Fluid后，Pod（无论是否访问数据集）的调度策略需要进行优化。

整个优化过程将分为三个阶段。

# 第一阶段——POC验证
## Goal
只考虑基本调度，不考虑dataset和runtime当前所处生命周期、node当前资源水位。

主要基于node的标签、Pod挂载的PVC等信息，通过webhook方式为Pod自动注入一些影响调度的字段。

## Background Knowledge
### 当前标签情况

Cache-worker所在的主机，均会打上三个标签：
* fluid.io/s-{$RuntimeType}-{$Namespace}-{$Name}:true
* fluid.io/s-{$Namespace}-{$Name}:true
* fluid-exclusive:{$Namespace}-{$Name}（独占模式下启用）

Dataset对应的PVC，会打上此标签：
* fluid.io/s-default-hbase: "true"

Alluxio-worker Pod，会打上标签：
* role: allxio-worker

Jindofs-worker Pod，会打上标签：
* role: jindofs-worker

可基于这些标签向Pod注入亲和性信息，影响Pod的调度。

### webhook实现调研

目前，实现webhook主要有以下几种方式：
1. 自己实现一个http server，从http request中接收body，反序列化后进行处理，将处理结果序列化后进行返回。部分项目会在此基础上进行进一步封装。
2. 基于controller-runtime框架进行开发在manager中手动进行webhook server的注册，controller-runtime框架会根据注册内容自动启动http server。自己只需实现Handle方法。具体实现可参考[controller-runtime提供的示例程序](https://github.com/kubernetes-sigs/controller-runtime/blob/master/examples/builtins/mutatingwebhook.go) 

部分项目会在此基础上进行进一步封装。例如[kruise](https://github.com/openkruise/kruise/blob/master/pkg/webhook/server.go) ，它不仅仅实现了Pod的注入，还为自己添加的所有CRD都开发了webhook，功能较为复杂。因此它将所有Handle组成了一个HashMap，通过handlerGates决定哪些Handle处于active状态
3. 使用kube-builder自动生成代码kube-builder自动生成的代码实际使用了controller-runtime框架，但由于需要为资源对象实现Defaulter方法，所以实际只能实现CRD的webhook，不能直接实现buildin资源对象的webhook。

## Design
### webhook
webhook server作为一个独立的程序，通过service暴露，需要实现高可用。

使用MutatingWebhook向用户提供Pod的自动注入功能，创建如下MutatingWebhookConfiguration：
```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
...
webhooks:
- name: my-webhook.example.com
  failurePolicy: Ignore
  rules:
  - operations: ["CREATE"]
    apiGroups: ["core"]
    apiVersions: ["v1", "v1beta1"]
    resources: ["pods"]
    scope: "Namespaced"
  namespaceSelector:
    matchLabels:
      Fluid-Injection: enabled
```

用户需要开启自动注入时，在ns打上标签“Fluid-Injection: enabled”，开启此ns中所有Pod的自动注入。

该ns下不需要自动注入的Pod，打上标签“Fluid-Injection: disabled”。

### PreferInterface

为了实现可扩展性，将可注入的preferr信息抽象成schedule plugins

插件需实现该接口：
```
type AffinityInterface interface{
    Handle(pod *v1.Pod, runtimeInfos []base.RuntimeInfo)
}
```
由于目前Pod没有任何label，无法直接判断Pod挂载了哪个数据集，只能通过以下方法：
1. 检查Pod挂载的PVC，依次去k8s里get这些PVC，如果发现PVC上有数据集对应标签，说明Pod挂载了这个数据集
2. 检查Pod挂载的PVC，依次和dataset集合比较，如果发现Name+Namespace相同，说明Pod挂载了这个数据集

目前计划使用方案1

确定Pod挂载数据集后，依次去get相应的runtime，构建[]base.runtimeInfo作为调度所需信息传递给插件

向插件直接传递Pod的指针。每个插件，都可以直接修改该Pod。
wehook按照设定的顺序串行调用所有插件，将最后修改得到的结果返回给apiserver。

由于webhook有超时时间，应控制插件的最长等待时间。


## Schedule Plugins
#### PreferNodesWithoutCache

主要针对不使用数据集的Pod
检查Pod挂载的PVC，如果发现PVC上没有数据集对应标签，则应优先往没有缓存的节点上调度：
```
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        weight: xxx
        - labelSelector:
            matchExpressions:
            - key: role
              operator: In
              values:
              - allxio-worker
              - jindofs-worker
            topologyKey: kubernetes.io/hostname
```
### PreferNodesWithCache
主要针对Fuse global模式使用数据集的Pod

Pod应优先往有缓存的节点上调度：
```
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        weight: xxx
        nodeSelectorTerms:
        - matchExpressions:
          - key: fluid.io/s-{$Namespace}-{$Name}:true
            operator: In
            values:
            - true
```

### RequireNodesWithFuse
主要针对Fuse 非global模式使用数据集的Pod。
由于部分非官方调度器可能不能保证将Pod调度到有Fuse的节点，需要注入require信息来保证调度

# 第二阶段
## Goal
调度时考虑缓存当前状态，例如：
1. alluxio/jindo内某个缓存信息
2. 节点内page cache缓存

## Scene
### 缩容（待优化）
如果某个fuse正在被访问，则该worker/fuse节点无法缩容成功。但如果用户仍在创建访问该数据集的Pod，可能会继续有新的调度过来。

前提：缓存缩容策略合理

因此，需要保证新创建的访问该数据集的Pod不能调度过来

缩容worker/fuse节点时，为节点打上标签data.fluid.io/storage-scalein-{$Namespace}-{$Dataset}:true

检查Pod挂载的PVC，如果发现挂载了此数据集，则为此Pod注入以下内容：
```
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: data.fluid.io/storage-scalein-{$Namespace}-{$Dataset}
            operator: NotIn
            values:
            - true
```

# 第三阶段
考虑数据集、Pod协同调度