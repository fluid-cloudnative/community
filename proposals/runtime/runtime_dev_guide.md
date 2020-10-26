# 增加新的Runtime支持

Fluid提供了Runtime接口，并且假设Runtime和Dataset是一对一的关系，开发者支持扩展不同的Runtime。具体操作如下：

1.安装kubebuilder

到 kubebuilder 的 [GitHub release 页面](https://github.com/kubernetes-sigs/kubebuilder/releases/tag/v2.1.0)上下载与您操作系统对应的 kubebuilder 安装包。

将下载好的安装包解压后将其移动到 /usr/local/kubebuilder 目录下，并将 /usr/local/kubebuilder/bin 添加到您的 $PATH 路径下。

2.下载最新的fluid代码并且创建新的branch

```
git clone https://github.com/fluid-cloudnative/fluid.git
cd fluid
git checkout -b jindo_runtime
```

3.利用生成工具产生JindoRuntime和JindoRuntimeController


```shell
kubebuilder create api --group data --version v1alpha1 --kind JindoRuntime --namespaced true
Create Resource [y/n]
y
Create Controller [y/n]
y
Writing scaffold for you to edit...
api/v1alpha1/jindoruntime_types.go
controllers/jindoruntime_controller.go
2020/10/25 16:21:06 error updating main.go: open main.go: no such file or directory
```

>  注意此处错误可以忽略

>  核心工作是定义JindoRuntime的Spec中的API

4.将`controllers/jindoruntime_controller.go`拷贝到`pkg/controllers/v1alpha1/jindo`

```
cd fluid
mv controllers/jindoruntime_controller.go pkg/controllers/v1alpha1/jindo
```

5.jindoruntime_controller.go的方法如下

```golang
/*

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package jindo

import (
	"context"

	"sync"

	"github.com/pkg/errors"

	"github.com/go-logr/logr"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/client-go/tools/record"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"

	datav1alpha1 "github.com/fluid-cloudnative/fluid/api/v1alpha1"
	"github.com/fluid-cloudnative/fluid/pkg/common"
	"github.com/fluid-cloudnative/fluid/pkg/controllers"
	"github.com/fluid-cloudnative/fluid/pkg/ddc/base"
	cruntime "github.com/fluid-cloudnative/fluid/pkg/runtime"
	"github.com/fluid-cloudnative/fluid/pkg/utils"
)

// Use compiler to check if the struct implements all the interface
var _ controllers.RuntimeReconcilerInterface = (*RuntimeReconciler)(nil)

// RuntimeReconciler reconciles a JindoRuntime object
type RuntimeReconciler struct {
	Scheme  *runtime.Scheme
	engines map[string]base.Engine
	mutex   *sync.Mutex
	*controllers.RuntimeReconciler
}

// NewRuntimeReconciler create controller for watching runtime custom resources created
func NewRuntimeReconciler(client client.Client,
	log logr.Logger,
	scheme *runtime.Scheme,
	recorder record.EventRecorder) *RuntimeReconciler {
	r := &RuntimeReconciler{
		Scheme:  scheme,
		mutex:   &sync.Mutex{},
		engines: map[string]base.Engine{},
	}
	r.RuntimeReconciler = controllers.NewRuntimeReconciler(r, client, log, recorder)
	return r
}

//Reconcile reconciles jindo runtime
// +kubebuilder:rbac:groups=data.fluid.io,resources=jindoruntimes,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=data.fluid.io,resources=jindoruntimes/status,verbs=get;update;patch

func (r *RuntimeReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
	ctx := cruntime.ReconcileRequestContext{
		Context:        context.Background(),
		Log:            r.Log.WithValues("jindoruntime", req.NamespacedName),
		NamespacedName: req.NamespacedName,
		Recorder:       r.Recorder,
		Category:       common.AccelerateCategory,
		RuntimeType:    runtimeType,
		Client:         r.Client,
		FinalizerName:  runtimeResourceFinalizerName,
	}

	ctx.Log.V(1).Info("process the request", "request", req)

	//	1.Load the Runtime
	runtime, err := r.getRuntime(ctx)
	if err != nil {
		if utils.IgnoreNotFound(err) == nil {
			ctx.Log.V(1).Info("The runtime is not found", "runtime", ctx.NamespacedName)
			return ctrl.Result{}, nil
		} else {
			ctx.Log.Error(err, "Failed to get the ddc runtime")
			return utils.RequeueIfError(errors.Wrap(err, "Unable to get ddc runtime"))
		}
	}
	ctx.Runtime = runtime
	ctx.Log.V(1).Info("process the runtime", "runtime", ctx.Runtime)

	// reconcile the implement
	return r.ReconcileInternal(ctx)
}

//SetupWithManager setups the manager with RuntimeReconciler
func (r *RuntimeReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&datav1alpha1.JindoRuntime{}).
		Complete(r)
}
```

5.开发Jindo Engine

5.1. 创建jindo engine的文件夹, jindo engine的实现需要参考alluxioRuntime的engine。

```
mkdir pkg/ddc/jindo
vi pkg/ddc/jindo/engine.go
```


这里涉及到两个问题： 

	1.如何初始化Engine，初始化Engine的方式可以参考`pkg/ddc/alluxio/engine.go`中的Build方法, 并且需要注册到`pkg/ddc/factory.go`  
	2.Engine的生命周期管理


5.2. 其中需要实现的接口Implement定义在`pkg/ddc/base/engine.go`, 具体需要实现的方法如下。

```golang
// The real engine should implement
type Implement interface {
	UnderFileSystemService
	// Is the master ready
	CheckMasterReady() (ready bool, err error)
	// are the workers ready
	CheckWorkersReady() (ready bool, err error)
	// ShouldSetupMaster checks if we need setup the master
	ShouldSetupMaster() (should bool, err error)
	// ShouldSetupWorkers checks if we need setup the workers
	ShouldSetupWorkers() (should bool, err error)

	ShouldCheckUFS() (should bool, err error)
	// setup the cache master
	SetupMaster() (err error)
	// setup the cache worker
	SetupWorkers() (err error)
	// check if it's Bound to the dataset
	// IsBoundToDataset() (bound bool, err error)
	// Bind to the dataset
	UpdateDatasetStatus(phase datav1alpha1.DatasetPhase) (err error)
	// Prepare the mounts and metadata if it's not ready
	PrepareUFS() (err error)
	// Shutdown and clean up the engine
	Shutdown() error

	// AssignNodesToCache picks up the nodes for replicas
	AssignNodesToCache(desiredNum int32) (currentNum int32, err error)
	// CheckRuntimeHealthy checks runtime healthy
	CheckRuntimeHealthy() (err error)
	// UpdateCacheOfDataset updates cache of the dataset
	UpdateCacheOfDataset() (err error)
	// CheckAndUpdateRuntimeStatus checks and updates the status
	CheckAndUpdateRuntimeStatus() (ready bool, err error)

	CreateVolume() error

	// SyncReplicas syncs the replicas
	SyncReplicas(ctx cruntime.ReconcileRequestContext) error

	// SyncMetadata syncs all metadata from UFS
	SyncMetadata() (err error)

	// Destroy the Volume
	DeleteVolume() (err error)
}

// UnderFileSystemService interface defines the interfaces that should be implemented
// by a underlayer fileSystem service for the data. The implementation is the underlayer file system connector.
// It is responsible for checking ufs and preload the data.
// Thread safety is required from implementations of this interface.
type UnderFileSystemService interface {
	UsedStorageBytes() (int64, error)

	FreeStorageBytes() (int64, error)

	TotalStorageBytes() (int64, error)

	TotalFileNums() (int64, error)
}

```









