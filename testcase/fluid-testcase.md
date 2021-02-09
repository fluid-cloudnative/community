# fluid测试用例

## 目录

- [1.安装fluid](#1-安装fluid)
  - [获得最新的fluid项目](#获得最新的fluid项目)
    - [无fluid仓库](#无fluid仓库)
    - [有fluid仓库](#有fluid仓库)
  - [卸载旧版本的fluid](#卸载旧版本的fluid)
  - [安装fluid](#安装fluid)
  - [检查fluid运行状态](#检查fluid运行状态)
- [2.创建Dataset](#2-创建Dataset)
  - [创建Dataset对应的yaml文件](#创建Dataset对应的yaml文件)
  - [创建Dataset](#创建Dataset)
- [3.创建AlluxioRuntime](#3-创建AlluxioRuntime)
  - [创建AlluxioRuntime对应的yaml文件](#创建AlluxioRuntime对应的yaml文件)
  - [创建AlluxioRuntime](#创建AlluxioRuntime)
  - [检查AlluxioRuntime对应的pod是否正常运行](#检查AlluxioRuntime对应的pod是否正常运行)
  - [检查PV和PVC是否正常创建](#检查PV和PVC是否正常创建)
- [4.判断Dataset是否bound](#4-判断Dataset是否bound)
  - [检查AlluxioRuntime是否Ready](#检查AlluxioRuntime是否Ready)
  - [检查Dataset是否bound](#检查Dataset是否bound)
- [5.删除Dataset](#5-删除Dataset)
- [6.删除AlluxioRuntime](#6-删除AlluxioRuntime)
- [7.DataLoad数据预加载](#7-DataLoad数据预加载)
  - [创建DataLoad对应的yaml文件](#创建DataLoad对应的yaml文件)
  - [创建DataLoad](#创建DataLoad)
  - [检查DataLoad是否已经正常运行](#检查DataLoad是否已经正常运行)
  - [检查DataLoad是否执行成功](#检查DataLoad是否执行成功)
  - [删除DataLoad](#删除DataLoad)
- [8.DataBackup数据备份](#8-DataBackup数据备份)
  - [备份到本地](#备份到本地)
    - [删除历史备份记录-local](#删除历史备份记录-local)
    - [创建DataBackup对应的yaml文件-local](#创建DataBackup对应的yaml文件-local)
    - [创建DataBackup-local](#创建DataBackup-local)
    - [检查DataBackup是否执行成功-local](#检查DataBackup是否执行成功-local)
  - [备份到PVC](#备份到PVC)
    - [创建PV和PVC](#创建PV和PVC)
    - [删除历史备份记录-pvc](#删除历史备份记录-pvc)
    - [创建DataBackup对应的yaml文件-pvc](#创建DataBackup对应的yaml文件-pvc)
    - [创建DataBackup-pvc](#创建DataBackup-pvc)
    - [检查DataBackup是否执行成功-pvc](#检查DataBackup是否执行成功-pvc)
  - [删除DataBackup](#删除DataBackup)

## 1-安装fluid

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

## 2-创建Dataset

该节展示了如何创建一个Dataset，以名字为spark的Dataset为例。

### 创建Dataset对应的yaml文件

dataset.yaml文件的内容可以根据需要进行编辑。

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

## 3-创建AlluxioRuntime

该节展示了如何创建Dataset对应的AlluxioRuntime，同样以spark为例。

### 创建AlluxioRuntime对应的yaml文件

runtime.yaml文件的内容可以根据需要进行编辑。

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

查看master，输入以下命令，如果master正常运行，返回`Running`。

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

### 检查PV和PVC是否正常创建

查看PV是否正常创建，输入以下命令，如果PV正常创建，返回`Bound`。

```shell
kubectl get pv | awk '$1=="spark" && $7=="fluid" {print $5}'
```

查看PVC是否正常创建，输入以下命令，如果PVC正常创建，返回`Bound`。

```shell
kubectl get pvc | awk '$1=="spark" && $3=="spark" && $6=="fluid" {print $2}'
```

如果以上条件都满足，说明fluid成功创建了PV和PVC。

## 4-判断Dataset是否bound

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

## 5-删除Dataset

输入以下命令删除Dataset。

```shell
kubectl delete dataset spark
```

检查Dataset是否被删除，输入以下命令输出空或者`No resources found in default namespace.`。

```shell
kubectl get dataset | awk '$1=="spark"'
```

检查AlluxioRuntime是否被删除，输入以下命令输出空或者`No resources found in default namespace.`。

```shell
kubectl get alluxioruntime | awk '$1=="spark"'
```

## 6-删除AlluxioRuntime

输入以下命令删除AlluxioRuntime。

```shell
kubectl delete alluxioruntime spark
```

检查AlluxioRuntime是否被删除，输入以下命令输出空或者`No resources found in default namespace.`。

```shell
kubectl get alluxioruntime | awk '$1=="spark"'
```

## 7-DataLoad数据预加载

本节展示了如何借助DataLoad进行数据预加载，同样以spark数据集为例。首先创建好相应的Dataset和AlluxioRuntime。

1. [创建Dataset](#2.创建Dataset)

2. [创建AlluxioRuntime](#3.创建AlluxioRuntime)

3. [判断Dataset是否bound](#4.判断Dataset是否bound)

> 需要注意的是，创建DataLoad进行数据预加载并不需要Dataset处于bound的状态，这里多一步判断Dataset是否bound只是可以为后续出错排查提供方便。

### 创建DataLoad对应的yaml文件

```shell
cat << EOF > dataload.yaml
apiVersion: data.fluid.io/v1alpha1
kind: DataLoad
metadata:
  name: spark-dataload
spec:
  dataset:
    name: spark
    namespace: default
EOF
```

### 创建DataLoad

输入以下命令创建DataLoad。

```shell
kubectl create -f dataload.yaml
```

检查DataLoad是否成功创建，如果创建成功，输入以下命令输出DataLoad的名字`spark-dataload`。

```shell
kubectl get dataload | awk '{print $1}' | grep ^spark-dataload$
```

### 检查DataLoad是否已经正常运行

检查DataLoad是否已经创建Job，成功创建Job输入以下命令输出不为空。

```shell
kubectl get job | awk '$1=="spark-dataload-loader-job"'
```

检查DataLoad对应的Pod的状态，正常运行应该返回`Running`。

```shell
kubectl get pod | grep ^spark-dataload-loader | awk '{print $3}'
```

检查DataLoad是否已经开始运行，输入以下命令返回`Pending`或者`Loading`，如果Dataset的size很小的话，加载数据操作会很快完成，也有可能返回`Complete`或者`Failed`，出现以上几种状态之一都能够说明DataLoad已经在正常运行了。

```shell
kubectl get dataload | awk '$1=="spark-dataload" {print $3}'
```

### 检查DataLoad是否执行成功

检查DataLoad创建的Job的状态，执行成功，输入以下命令返回`Complete`，执行失败，返回`Failed`。如果Job未执行结束，返回空。

```shell
kubectl get job spark-dataload-loader-job -o jsonpath={.status.conditions[0].type}
```

检查DataLoad是否执行成功，如果执行成功，输入以下命令返回`Complete`，如果失败，返回`Failed`。如果DataLoad未结束，输出`Pending`或者`Loading`。

```shell
kubectl get dataload | awk '$1=="spark-dataload" {print $3}'
```

检查Dataset的数据是否被缓存，输入以下命令返回缓存比例，正常情况下，在有缓存能力的情况下，缓存比例应该大于0。本例中，DataLoad执行结束后输出`100.0%`。

```shell
kubectl get dataset | awk '$1=="spark" {print $5}'
```

如果以上命令都是成功状态，说明DataLoad执行成功且正常加载了数据，否则说明加载数据失败。

### 删除DataLoad

输入以下命令删除DataLoad。

```shell
kubectl delete dataload spark-dataload
```

检查DataLoad是否删除，如果DataLoad已经正常删除，输入以下命令输出为空或者`No resources found in default namespace.`。

```shell
kubectl get dataload | awk '$1=="spark-dataload"'
```

检查DataLoad创建的Job是否删除，如果DataLoad已经正常删除，输入以下命令输出为空或者`No resources found in default namespace.`。

```shell
kubectl get job | awk '{print $1}' | grep ^spark-dataload-loader-job
```

如果以上条件都满足，说明DataLoad已经成功删除。需要注意的是，当DataLoad对应的Dataset删除之后，DataLoad会被级联删除。

## 8-DataBackup数据备份

该节展示了如何借助DataBackup对Dataset进行数据备份，同样以spark数据集为例。首先要创建Dataset和对应的AlluxioRuntime，同时确保Dataset已经处于bound状态。

1. [创建Dataset](#2.创建Dataset)

2. [创建AlluxioRuntime](#3.创建AlluxioRuntime)

3. [判断Dataset是否bound](#4.判断Dataset是否bound)

### 备份到本地

#### 删除历史备份记录-local

这里将备份文件保存到`/root/hhj/backup`文件夹下，为了方便后续验证备份结果，`/root/hhj/backup`文件夹最好是一个空的文件夹或者里面没有不存在对同名数据集（spark）的备份记录。可以依次输入以下命令删除名字为spark的数据集的备份记录。

```shell
rm -f /root/hhj/backup/metadata-backup-spark-default.gz
rm -f /root/hhj/backup/spark-default.yaml
```

检查相关备份记录是否删除，输入以下命令输出为空。

```shell
ls /root/hhj/backup | awk '$1=="metadata-backup-spark-default.gz" || $1=="spark-default.yaml" {print $1}'
```

#### 创建DataBackup对应的yaml文件-local

backup-local.yaml文件的内容可以根据需要进行编辑。

```shell
cat << EOF > backup-local.yaml
apiVersion: data.fluid.io/v1alpha1
kind: DataBackup
metadata:
  name: spark-backup-local
spec:
  dataset: spark
  backupPath: local:///root/hhj/backup
EOF
```

#### 创建DataBackup-local

输入以下命令创建DataBackup。

```shell
kubectl create -f backup-local.yaml
```

检查DataBackup是否成功创建，输入以下命令输出`spark-backup-local`。

```shell
kubectl get databackup | awk '{print $1}' | grep ^spark-backup-local$
```

#### 检查DataBackup是否执行成功-local

检查DataBackup对应的Pod是否执行成功，输入以下命令输出`Completed`。如果输出`Running`说明备份工作仍在执行，如果输出其它错误状态，说明DataBackup运行失败。

```shell
kubectl get pod | awk '$1=="spark-backup-local-pod" {print $3}'
```

检查DataBackup是否执行成功，输入以下命令输出`Complete`。如果输出`Pending`或者`Backuping`说明备份工作仍在执行，如果输出其它错误状态，说明DataBackup运行失败。

```shell
kubectl get databackup | awk '$1=="spark-backup-local" {print $3}'
```

检查是否成功生成了用于后续恢复的备份文件，检查设置的备份路径下是否生辰了对应的文件，输入以下命令输出`metadata-backup-spark-default.gz`。

```shell
ls /root/hhj/backup | awk '$1=="metadata-backup-spark-default.gz" {print $1}'
```

输入以下命令输出`spark-default.yaml`。

```shell
ls /root/hhj/backup | awk '$1=="spark-default.yaml" {print $1}'
```

如果以上条件都满足，说明DataBackup执行成功，完成了对spark数据集的备份。

### 备份到PVC

#### 创建PV和PVC

这里使用nfs创建PV和PVC用于测试，首先编写创建PV、PVC的yaml文件。注意，创建的PV和PVC一定要有写的权限，也就是它们的accessModes字段必须为`ReadWriteOnce`或者`ReadWriteMany`。

```shell
cat << EOF > nfs.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-imagenet
spec:
  capacity:
    storage: 150Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  mountOptions:
  - vers=3
  - nolock
  - proto=tcp
  - rsize=1048576
  - wsize=1048576
  - hard
  - timeo=600
  - retrans=2
  - noresvport
  - nfsvers=4.1
  nfs:
    path: /
    server: 38037492dc-pol25.cn-shanghai.nas.aliyuncs.com
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-imagenet
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 150Gi
  storageClassName: nfs
EOF
```

输入以下命令创建PV和PVC。

```shell
kubectl create -f nfs.yaml
```

检查PV是否创建并且和PVC已经bound，输入以下命令输出`Bound`。

```shell
kubectl get pv | awk '$1=="nfs-imagenet" {print $5}'
```

检查PVC是否创建并且和PV已经bound，输入以下命令输出`Bound`。

```shell
kubectl get pvc | awk '$1=="nfs-imagenet" {print $2}'
```

如果以上条件都满足，说明用于测试的PV和PVC已经创建。

#### 删除历史备份记录-pvc

为了方便观测结果和操作PVC存储中的内容，可以把nfs挂在至本地的某个文件夹下，这里我以挂载到`/mnt/nfs`文件夹下为例，备份文件在PVC存储路径为`/backup`，也就对应本地的`/mnt/nfs/backup`文件夹。为了方便后续验证备份结果，PVC中的`/backup`文件夹最好是一个空的文件夹或者里面没有不存在对同名数据集（spark）的备份记录。可以依次输入以下命令删除名字为spark的数据集的备份记录。

```shell
rm -f /mnt/nfs/backup/metadata-backup-spark-default.gz
rm -f /mnt/nfs/backup/spark-default.yaml
```

检查相关备份记录是否已经删除，删除成功的话输入以下命令输出空。

```shell
ls /root/hhj/backup | awk '$1=="metadata-backup-spark-default.gz" || $1=="spark-default.yaml" {print $1}'
```

#### 创建DataBackup对应的yaml文件-pvc

backup-local.yaml文件的内容可以根据需要进行编辑。

```shell
cat << EOF > backup-pvc.yaml
apiVersion: data.fluid.io/v1alpha1
kind: DataBackup
metadata:
  name: spark-backup-pvc
spec:
  dataset: spark
  backupPath: pvc://nfs-imagenet/backup/                                     
EOF
```

#### 创建DataBackup-pvc

输入以下命令创建DataBackup。

```shell
kubectl create -f backup-pvc.yaml
```

检查DataBackup是否成功创建，输入以下命令输出`spark-backup-local`。

```shell
kubectl get databackup | awk '{print $1}' | grep ^spark-backup-pvc$
```

#### 检查DataBackup是否执行成功-pvc

检查DataBackup对应的Pod是否执行成功，输入以下命令输出`Completed`。如果输出`Running`说明备份工作仍在执行，如果输出其它错误状态，说明DataBackup运行失败。

```shell
kubectl get pod | awk '$1=="spark-backup-pvc-pod" {print $3}'
```

检查DataBackup是否执行成功，输入以下命令输出`Complete`。如果输出`Pending`或者`Backuping`说明备份工作仍在执行，如果输出其它错误状态，说明DataBackup运行失败。

```shell
kubectl get databackup | awk '$1=="spark-backup-pvc" {print $3}'
```

检查是否成功生成了用于后续恢复的备份文件，检查设置的备份路径下是否生辰了对应的文件，输入以下命令输出`metadata-backup-spark-default.gz`。

```shell
ls /mnt/nfs/backup | awk '$1=="metadata-backup-spark-default.gz" {print $1}'
```

输入以下命令输出`spark-default.yaml`。

```shell
ls /mnt/nfs/backup | awk '$1=="spark-default.yaml" {print $1}'
```

如果以上条件都满足，说明DataBackup执行成功，完成了对spark数据集的备份。

### 删除DataBackup

输入以下命令删除DataBackup，这里以删除`spark-backup-local`为例。

```shell
kubectl delete databackup spark-backup-local
```

检查是否已经删除DataBackup，输入以下命令输出为空或者`No resources found in default namespace.`。

```shell
kubectl get databackup | awk '$1=="spark-backup-local"'
```



