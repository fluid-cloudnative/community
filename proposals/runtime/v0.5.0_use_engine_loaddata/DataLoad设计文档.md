# DataLoad设计文档

## 1. Reconcile()逻辑

1. 初始化ReconcileRequestContext的部分属性，记为ctx。
2. 获得DataLoad，记为targetDataload。
3. 查看targetDataload是否有DeletionTimeStamp，如果有，进行DataLoad的删除。
4. 获得要进行dataload的Dataset，记为targetDataset，同时初始化ctx的Dataset和NamespacedName。

> 这里注意，ctx的NamespacedName是与Dataset一致的，而不是与DataLoad一致的。这是因为ctx要用于创建Engine，Engine是与某个Runtime相关的，Engine的NamespacedName要与Runtime一致才能保证Engine的一些方法的正常使用，这也就要求用于创建Engine的ctx的NamespacedName要和Runtime一致，即和Dataset一致。这与之前的设计有所不同。

5. 获取Runtime。分为两步：

> - 获取Runtime的类型。在这之前会判断Runtime是否已经bound。
> - 根据Runtime的类型创建不同的Runtime，初始化ctx的Runtime。

6. 添加finalizer。
7. 添加owner。
8. 创建或者获得Engine，方法如下：

```go
// GetOrCreateEngine gets the Engine
func (r *DataLoadReconciler) GetOrCreateEngine(
	ctx cruntime.ReconcileRequestContext) (engine base.Engine, err error) {
	found := false
	id := ddc.GenerateEngineID(ctx.NamespacedName)
	r.mutex.Lock()
	defer r.mutex.Unlock()
	if engine, found = r.engines[id]; !found {
		engine, err = ddc.CreateEngine(id,
			ctx)
		if err != nil {
			return nil, err
		}
		r.engines[id] = engine
		r.Log.V(1).Info("Put Engine to engine map")
	} else {
		r.Log.V(1).Info("Get Engine from engine map")
	}

	return engine, err
}
```

9. 最后，reconcile DataLoad，参数有ctx、要reconcile的DataLoad以及engine：

```go
return r.ReconcileDataLoad(ctx, targetDataload, engine)
```

## 2. ReconcileDataLoad()逻辑

对于一个DataLoad，它的Status.Phase字段记录该DataLoad当前的状态，控制器根据DataLoad所处的不同的Phase，采取不同的动作。Phase之间转换的次序为`None -> Pending -> Loading -> Loaded or Failed`，目前的PR与之前DataLoad的状态机的设计是一致的。下面依次介绍各个Phase中控制器所做的操作，会重点讲述改动的部分。

### 2.1 reconcileNoneDataLoad

该阶段主要是将DataLoad.Status.Conditions初始化，然后将DataLoad的Phase由None转为Pending。

```go
dataloadToUpdate.Status.Phase = cdataload.DataLoadPhasePending
if len(dataloadToUpdate.Status.Conditions) == 0 {
	dataloadToUpdate.Status.Conditions = []datav1alpha1.DataLoadCondition{}
}
```

### 2.2 reconcilePendingDataLoad

该阶段所作的操作如下，下面把要进行load data操作的Dataset称为targetDataset：

- 检查targetDataset是否存在。如果不存在则重新reconcile，否则往下执行。
- 检查targetDataset上是否正被其它的DataLoad占用。如果被占用则重新reconcile，否则往下执行。
- 检查Runtime是否ready，通过调用Engine的Ready()方法。如果未ready则重新reconcile，否则往下执行。

> Engine.Ready()方法是新增的方法，该方法是通过调用Implement的CheckMasterReady()和CheckWorkersReady()来实现的。替代了之前用case判断的方式，以后新增Runtime不会影响到该部分的代码。如果Runtime的master和workers都就绪的话，Engine.Ready()返回true，否则返回false。
>
> **pkg/ddc/base/load_data.go**
>
> ```go
> // Check if the runtime is ready
> func (t *TemplateEngine) Ready() (ready bool, err error) {
> 	masterReady, err := t.Implement.CheckMasterReady()
> 	if err != nil {
> 		t.Log.Error(err, "Failed to check if the master is ready.")
> 		return ready, err
> 	}
> 	workersReady, err := t.Implement.CheckWorkersReady()
> 	if err != nil {
> 		t.Log.Error(err, "Failed to check if the workers are ready.")
> 		return ready, err
> 	}
> 	ready = masterReady && workersReady
> 	return ready, nil
> }
> ```

- 将targetDataset.Status.DataLoadRef设为当前DataLoad，表示targetDataset已经被当前DataLoad占用，不能够再进行除该DataLoad之外的加载数据操作。设置成功则往下继续执行，设置失败则重新reconcile。
- 更新DataLoad的Phase为loading。

### 2.3 reconcileLoadingDataLoad

该阶段所作的操作如下：

- 启动加载数据的job，通过调用Engine.LoadData()方法实现。

```go
releaseName, jobName, err := engine.LoadData(ctx, targetDataload)
if err != nil {
	return utils.RequeueIfError(err)
}
```

> Engine.LoadData()也是新增的方法，该方法通过调用Implement的LoadData()方法实现。
>
> **pkg/ddc/base/load_data.go**
>
> ```go
> // Load the data
> func (t *TemplateEngine) LoadData(ctx cruntime.ReconcileRequestContext, targetDataload datav1alpha1.DataLoad) (string, string, error) {
> 	return t.Implement.LoadData(ctx, targetDataload)
> }
> ```
>
> 不像之前那样，创建release、job的逻辑都写在DataLoad的controller中，现在加载数据的功能由Engine提供，DataLoad的controller只需要调用方法即可，不同类型的Runtime可以在底层有不同实现，但对DataLoad是透明的。
>
> AlluxioEngine加载数据的逻辑与之前设计基本一致，分为以下几步：
>
> - 检查helm release是否存在（job是通过helm创建的）。如果存在直接返回job和release的名字，不再重复创建，否则继续往下执行。
> - 根据DataLoad安装helm chart，安装完成后返回job和release的名字。

- 检查Engine创建出的DataLoad的job的运行状态。
  - 如果job未找到，helm release存在的话，删除helm release并且重新reconcile，相当于重新reconcileLoadingDataLoad，重新创建DataLoad任务。
  - 如果job存在的话，检查它的运行状态。如果job已经Failed或者Complete，那么在DataLoad中记录下job对应的状态，同时把DataLoad的Phase更新为Failed或者Complete状态。否则的话，说明job未执行结束，重新reconcile。

### 2.4 reconcileFailedDataLoad()和reconcileLoadedDataLoad()

对于Failed和Complete状态的DataLoad，所作的操作目前是一致的，都会将自己所占的Dataset释放，也就是将Dataset.Status.DataLoadRef字段设置为空串，保证在当前DataLoad结束之后，其它DataLoad可以继续执行。释放Dataset后不再requeue。

```go
err := r.releaseLockOnTargetDataset(ctx, targetDataload, log)
	if err != nil {
		log.Error(err, "can't release lock on target dataset", "targetDataset", targetDataload.Spec.Dataset)
		return utils.RequeueIfError(err)
	}

	// 2. record event and no requeue
	log.Info("DataLoad Loaded, no need to requeue")
	jobName := utils.GetDataLoadJobName(utils.GetDataLoadReleaseName(targetDataload.Name))
	r.Recorder.Eventf(&targetDataload, v1.EventTypeNormal, common.DataLoadJobComplete, "DataLoad job %s succeeded", jobName)
	return utils.NoRequeue()
```

## 3. 关于DataLoad操作的幂等性

当前设计下，对于DataLoad操作的幂等性，我的理解如下：

- 如果是同一个DataLoad因某种原因中断后再执行的话，controller会根据DataLoad的Phase进行reconcile。由于再次reconcile时会直接从DataLoad当前Phase往下执行，当DataLoad中断重启后，它之前经历的Phase的某些条件已经不满足时，是会出现问题的，因为DataLoad没有重新从None的Phase执行，可能会发生到达不了当前Phase的风险。
- 如果是对同一个Dataset重复进行相同的DataLoad操作，那么效果应该是与底层Runtime提供的LoadData的机制相关的。以Alluxio为例，Alluxio的DataLoad的实现底层是运行了`alluxio fs distributedLoad ...`命令。在测试中，重复执行DataLoad操作，在执行过一次后，后面的job都会很快到达Complete状态，说明`alluxio fs distributedLoad ...`命令对于已经加载的数据很可能是不会重新再去加载的。