# fluid测试用例

## 目录

- [1.安装fluid](#1.安装fluid)
  - [获得最新的fluid项目](#获得最新的fluid项目)
    - [无fluid仓库](#无fluid仓库)
    - [有fluid仓库](#有fluid仓库)
  - [卸载旧版本的fluid](#卸载旧版本的fluid)
  - [安装fluid](#安装fluid)
  - [检查fluid运行状态](#检查fluid运行状态)
- [2.创建Dataset](#2.创建Dataset)
  - [创建Dataset对应的yaml文件](#创建Dataset对应的yaml文件)
  - [创建Dataset](#创建Dataset)
- [3.创建AlluxioRuntime](#3.创建AlluxioRuntime)
  - [创建AlluxioRuntime对应的yaml文件](#创建AlluxioRuntime对应的yaml文件)
  - [创建AlluxioRuntime](#创建AlluxioRuntime)
  - [检查AlluxioRuntime对应的pod是否正常运行](#检查AlluxioRuntime对应的pod是否正常运行)
- [4.判断Dataset是否bound](#4.判断Dataset是否bound)
  - [检查AlluxioRuntime是否Ready](#检查AlluxioRuntime是否Ready)
  - [检查Dataset是否bound](#检查Dataset是否bound)
- [5.删除Dataset](#5.删除Dataset)
- [6.删除AlluxioRuntime](#6.删除AlluxioRuntime)

## 1.安装fluid

### 获得最新的fluid项目

如果测试环境没有fluid的仓库，则需要下载，如果已经有了fluid的仓库，则直接git pull对应的分支即可。

#### 无fluid仓库

进入需要保存fluid仓库的目录，这里以放在根目录下为例，获取fluid项目的源码。

```bash
git clone https://github.com/fluid-cloudnative/fluid.git /fluid
```

#### 有fluid仓库

进入fluid仓库。

```shell
cd /fluid
```

拉取fluid对应分支的最新的版本，这里以master分支为例。

```shell
git checkout master
git pull origin master:master
```

### 卸载旧版本的fluid

首先删除fluid创建的crd。

```shell
kubectl delete crd $(kubectl get crd | grep data.fluid.io | awk '{print $1}')
```

检查成功删除crd，输入以下命令输出为空。

```shell
kubectl get crd | grep data.fluid.io
```

然后卸载旧的fluid。

```shell
helm delete fluid
```

检查成功删除fluid，输入以下命令输出为空。

```shell
helm list | awk '{print $1}' | grep ^fluid$
```

### 安装fluid

创建命名空间。

```shell
kubectl create ns fluid-system
```

输入以下命令安装fluid。

```shell
helm install fluid /fluid/charts/fluid/fluid/
```

### 检查fluid运行状态

检查fluid安装成功，输入以下命令输出`fluid`。

```shell
helm list | awk '{print $1}' | grep ^fluid$
```

检查alluxioruntime-controller的运行状态，如果alluxioruntime-controller运行正常，返回`Running`。

```shell
kubectl get pod -n fluid-system | grep alluxioruntime-controller | awk '{print $3}'
```

检查dataset-controller的运行状态，如果dataset-controller运行正常，返回`Running`。

```shell
kubectl get pod -n fluid-system | grep dataset-controller | awk '{print $3}'
```

检查csi-nodepulgin的运行状态，如果csi-nodeplugin运行正常，返回`Running`，`Running`的个数与节点个数一致。

```shell
kubectl get pod -n fluid-system | grep csi-nodeplugin | awk '{print $3}'
```

如果以上结果返回都正确，说明fluid在正常运行。

## 2.创建Dataset

该节展示了如何创建一个Dataset，以名字为spark的Dataset为例。

### 创建Dataset对应的yaml文件

```shell
cat << EOF > dataset.yaml
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: spark
spec:
  mounts:
    - mountPoint: https://mirror.bit.edu.cn/apache/spark/
      name: spark
EOF
```

### 创建Dataset

输入以下命令创建Dataset。

```shell
kubectl create -f dataset.yaml
```

检查Dataset是否成功创建，输入以下命令输出`spark`。

```shell
kubectl get dataset | awk '{print $1}' | grep ^spark$
```

## 3.创建AlluxioRuntime

该节展示了如何创建Dataset对应的AlluxioRuntime，同样以spark为例。

### 创建AlluxioRuntime对应的yaml文件

```shell
cat << EOF > runtime.yaml 
apiVersion: data.fluid.io/v1alpha1
kind: AlluxioRuntime
metadata:
  name: spark
spec:
  replicas: 1
  tieredstore:
    levels:
      - mediumtype: MEM
        path: /dev/shm
        quota: 4Gi
        high: "0.95"
        low: "0.7"
  properties:
    alluxio.user.block.size.bytes.default: 256MB
    alluxio.user.streaming.reader.chunk.size.bytes: 256MB
    alluxio.user.local.reader.chunk.size.bytes: 256MB
    alluxio.worker.network.reader.buffer.size: 256MB
    alluxio.user.streaming.data.timeout: 300sec
  fuse:
    args:
      - fuse
      - --fuse-opts=kernel_cache,ro,max_read=131072,attr_timeout=7200,entry_timeout=7200,nonempty,max_readahead=0
EOF
```

### 创建AlluxioRuntime

输入以下命令创建AlluxioRuntime。

```shell
kubectl create -f runtime.yaml
```

检查Dataset是否成功创建，输入以下命令输出`spark`。

```shell
kubectl get alluxioruntime | awk '{print $1}' | grep ^spark$
```

### 检查AlluxioRuntime对应的pod是否正常运行

查看master，输入以下命令，如果master正常运行，返回`Running`

```shell
kubectl get pod | grep spark-master | awk '{print $3}'
```

查看worker，输入以下命令，如果workers正常运行，返回`Running`，`Running`的个数应该和AlluxioRuntime.spec.replicas属性一致。

```shell
kubectl get pod | grep spark-worker | awk '{print $3}'
```

查看fuse，输入以下命令，如果fuse正常运行，返回`Running`，`Running`的个数应该和AlluxioRuntime.spec.replicas属性一致。

```shell
kubectl get pod | grep spark-fuse | awk '{print $3}'
```

如果以上条件都满足，说明AlluxioRuntime正常运行。

## 4.判断Dataset是否bound

### 检查AlluxioRuntime是否Ready

检查Master是否Ready，Ready返回`Ready`。

```shell
kubectl get alluxioruntime | awk '$1=="spark"{print $2}'
```

检查Worker是否Ready，Ready返回`Ready`。

```shell
kubectl get alluxioruntime | awk '$1=="spark"{print $3}'
```

检查Fuse是否Ready，Ready返回`Ready`。

```shell
kubectl get alluxioruntime | awk '$1=="spark"{print $4}'
```

如果以上条件都满足，说明AlluxioRuntime已经Ready。

### 检查Dataset是否bound

检查Dataset是否bound，bound返回`Bound`。

```shell
kubectl get dataset | awk '$1=="spark"{print $6}'
```

## 5.删除Dataset

输入以下命令删除Dataset。

```shell
kubectl delete dataset spark
```

检查Dataset是否被删除，输入以下命令输出空。

```shell
kubectl get dataset | awk '$1=="spark"'
```

检查AlluxioRuntime是否被删除，输入以下命令输出空。

```shell
kubectl get alluxioruntime | awk '$1=="spark"'
```

## 6.删除AlluxioRuntime

输入以下命令删除AlluxioRuntime。

```shell
kubectl delete alluxioruntime spark
```

检查AlluxioRuntime是否被删除，输入以下命令输出空。

```shell
kubectl get alluxioruntime | awk '$1=="spark"'
```