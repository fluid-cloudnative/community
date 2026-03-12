# Proposal: Extend cache runtime interface for full data lifecycle

## 背景

Fluid 提供一种通用的数据引擎缓存系统的接入机制（CacheRuntime），降低非云原生领域数据引擎开发人员将其数据引擎接入到云原生环境中的学习和开发成本。目前仅实现了CRD的定义和基本的Reconcile逻辑，并未与实际的缓存系统（如Cuvine/Alluxio）进行联调测试。

此外，当前的 Cache Runtime 接口缺乏对端到端的数据操作和动态 runtime 更改的支持，这迫使用户不得不删除并重新创建数据集以进行日常维护（例如，引擎升级或故障恢复）。这导致了不必要的停机时间、运营开销和糟糕的用户体验。通过扩展接口以管理完整的数据生命周期，并支持就地升级和重建，Fluid能够提供无缝、有弹性和高效的数据管理，从而提高系统可用性，满足云原生数据编排的生产级要求。

## 现有问题

- 当前的Generic Cache Runtime 并未与缓存系统（Curvine等）进行联调，缺乏Curvine Mount的接口定义，POC 工作及测试用例有待完成；
- 当前的Generic Cache Runtime 并不支持DataLoad, DataProcess等数据操作，需要定义标准的 API；
- 当前的Generic Cache Runtime 并不支持 In-Place upgrade and cache rebuild。

## 目标

- 通用的缓存运行时接口能够快速将新引擎集成到 Fluid 中，使平台更具可扩展性和对操作员更友好。
- 完整的数据生命周期并支持就地升级和重建，Fluid 可以提供无缝、弹性和高效的数据管理，提高系统可用性，减少运营摩擦，并符合云原生数据编排的工业级要求。

## 方案

### 1. Generic Cache Runtime 集成 Curvine

Curvine 是 Master-Worker 架构，其示例配置如下：

```toml
# master configuration
[master]
meta_dir = "testing/meta"

# masta ha raft configuration.
[journal]
journal_addrs = [
    {id = 1, hostname = "master-sts-0.master-sts-svc", port = 8996}
]
journal_dir = "testing/journal"

# Worker configuration
[worker]
dir_reserved = "0"
data_dir = [
    "[DISK]testing/data",
]

[fuse]
mnt_path=/runtime-mnt/fuse
```

其 master / worker / fuse 的启动命令如下：

```shell
# Master /entrypoint.sh master start
/app/curvine/lib/curvine-server --service master --conf curvine-cluster.toml
# Worker /entrypoint.sh master stop
/app/curvine/lib/curvine-server --service worker --conf curvine-cluster.toml
# Fuse
/app/curvine/lib/curvine-fuse --conf curvine-cluster.toml
```

Cache Runtime 为缓存系统提供 RuntimeConfig，因此Curvine 组件需要**封装原始镜像的启动命令，先进行参数解析生成配置文件，再使用原始的启动命令启动进程**。

此外，Master Sts 启动后，需要执行 cv mount 进行挂载远程存储。

- 不同于curvine，Juicefs 是无Master架构，在[启动时就执行 format ](https://juicefs.com/docs/zh/community/getting-started/for_distributed/#4-创建文件系统)，只支持一个远程存储，因此直接在 worker/fuse 的启动命令里执行；

扩展 CacheRuntimeClass 定义，增加对 mount UFS 的支持

```go
type CacheRuntimeClass struct {
    // 当前 Cache System 支持哪些数据操作
    LifeCycleHook *LifeCycleHook `json:"lifeCycleHook,omitempty"`
}
type LifeCycleHook struct { 
    // 挂载 UFS 的 hook 操作，针对 Master-Slave 架构，需要在 Master 中执行
    MountUfs *MountUfs  `json:"mountUfs,omitempty"`
}
type MountUFS struct {
    // 执行的命令，必选，会在 Master Pod 中执行
    Command string `json:"command"`
    
    // 执行命令的超时时间（单位：秒），最小值为 5s.
    Timeout int `json:"timeout,omitempty"`
}
```

Curvine Cache Runtime 的处理流程如下图所示：本项工作的重点在于：

1. 提供启动脚本，将 Fluid 提供的 RuntimeConfig 转化为 Curvine 所使用的配置文件；
   - 拟采用 go template 要求的格式定义 Curvine 的配置文件，并进行替换；
1. 添加 Mount UFS 步骤，在 Master Sts 启动完成后，进入 Master Pod 执行 cv mount 操作；
   - **mount 操作在 CacheRuntimeClass 中定义，指定在特定的角色（如Master）的Pod 中执行指定的命令，以RuntimeConfig文件为参数。**
   - 对于 JuiceFS 缓存系统，不需要单独执行 mount 参数，在 CacheRuntimeClass 中不定义即可；

![img](./pics/curvine_integration.jpeg)

### 2. 定义标准API，支持 DataOperation

针对 DataOperation，扩展 CacheRuntimeClass 定义，表明支持哪些数据操作：

```go
type CacheRuntimeClass struct {
    // 当前 Cache System 支持哪些数据操作
    DataOperation	[]DataOperationSpec `json:"dataOperationSpec,omitempty"`
}

type DataOperationSpec struct {
    // Data Operation，如 DataLoad, DataBackup, DataMigration等
    // +kubebuilder:validation:Enum=DataLoad
    Name string `json:"name,string"`

    // The image name for DataOperation executing
    Image string `json:"name,string"`
    
    // Command for image container
    Command []string `json:"command,omitempty"`
    
    // Args for image container
    Args []string `json:"args,omitempty" 
}
```

Fluid 当前针对不同的 DataOperation 已经定义了 Engine 的接口。因此CacheRuntime 的 CacheEngine，需要实现下面的接口，完成相应的数据操作以及状态的流转。

```go
type DataOperator interface {
	Operate(ctx cruntime.ReconcileRequestContext, opStatus *datav1alpha1.OperationStatus, operation dataoperation.OperationInterface) (ctrl.Result, error)
}
```

为了保证与 TemplateEngine 的状态及处理逻辑一致，仍使用五种状态（None/Pending/Executing/Complete/Failed），其状态转换逻辑如下：

<img src="pics/state_transform.jpeg" alt="img" style="zoom:80%;" />

在核心实现上，与 TemplateEngine 的不同点在于 Helm 文件的生成，即需要实现接口

```go
// DataOperatorYamlGenerator is the implementation of DataOperator interface for runtime engine.
type DataOperatorYamlGenerator interface {
	GetDataOperationValueFile(ctx cruntime.ReconcileRequestContext, operation dataoperation.OperationInterface) (valueFileName string, err error)
}
```

针对 operation 的 不同 Type，做不同的处理。DataProcess 不区分缓存系统，可以复用现有的 Helm Yaml 生成逻辑；而其它的 DataOperation 都需要相应的缓存系统镜像及其配置。

- 通过新增的 DataOperationSpec 定义相应 Pod ，启动并执行相应的数据操作的命令，其中 Fluid DataOperation的相关配置信息，会挂载到 /etc/fluid/config/dataop 文件中；


### 3. 支持 In-Place Upgrade 和 ReBuild

版本更新时的原地升级：



配置更新时的缓存重建：



