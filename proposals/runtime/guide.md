
0.安装kube builder

```

```


1.利用生成工具产生JindoRuntime和JindoRuntimeController


```
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

2.

```

```


3.核心方法

```
func (r *RuntimeReconciler) ReconcileRuntime(engine base.Engine, ctx cruntime.ReconcileRequestContext) (ctrl.Result, error) {
```


