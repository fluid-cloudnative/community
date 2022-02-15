# fluid测试用例

## 目录

- [fluid测试用例](#fluid测试用例)
  - [目录](#目录)
  - [1-安装fluid](#1-安装fluid)
    - [获得最新的fluid项目](#获得最新的fluid项目)
      - [无fluid仓库](#无fluid仓库)
      - [有fluid仓库](#有fluid仓库)
    - [卸载旧版本的fluid](#卸载旧版本的fluid)
    - [安装fluid](#安装fluid)
    - [检查fluid运行状态](#检查fluid运行状态)
  - [2-使用Dataset访问WebUFS](#2-使用dataset访问webufs)
    - [创建Dataset和AlluxioRuntime](#创建dataset和alluxioruntime)
    - [检查Dataset和AlluxioRuntime状态](#检查dataset和alluxioruntime状态)
    - [通过Dataset访问数据](#通过dataset访问数据)
    - [环境清理](#环境清理)
  - [3-DataLoad数据预加载](#3-dataload数据预加载)
    - [创建DataLoad](#创建dataload)
    - [检查DataLoad状态](#检查dataload状态)
    - [检查DataLoad是否执行成功](#检查dataload是否执行成功)
    - [环境清理](#环境清理-1)
  - [4-DataBackup数据备份](#4-databackup数据备份)
    - [备份到本地](#备份到本地)
      - [删除历史备份记录-local](#删除历史备份记录-local)
      - [创建DataBackup对应的yaml文件-local](#创建databackup对应的yaml文件-local)
      - [创建DataBackup-local](#创建databackup-local)
      - [检查DataBackup是否执行成功-local](#检查databackup是否执行成功-local)
    - [备份到PVC](#备份到pvc)
      - [创建PV和PVC](#创建pv和pvc)
      - [删除历史备份记录-pvc](#删除历史备份记录-pvc)
      - [创建DataBackup对应的yaml文件-pvc](#创建databackup对应的yaml文件-pvc)
      - [创建DataBackup-pvc](#创建databackup-pvc)
      - [检查DataBackup是否执行成功-pvc](#检查databackup是否执行成功-pvc)
    - [删除DataBackup](#删除databackup)
  - [5-同一runtime类型条件下不同并发场景使用Patch进行更新节点标签](#5-同一runtime类型条件下不同并发场景使用patch进行更新节点标签)
    - [准备工作](#准备工作)
    - [多数据集并发调度到同一节点](#多数据集并发调度到同一节点)
    - [多数据集在同一节点并发调度与删除](#多数据集在同一节点并发调度与删除)
    - [多数据集在同一节点并发删除](#多数据集在同一节点并发删除)
  - [6-Fuse挂载点自动恢复](#6-fuse挂载点自动恢复)
    - [准备工作](#准备工作-1)
    - [创建Dataset和AlluxioRuntime](#创建dataset和alluxioruntime-1)
    - [为Namespace开启Webhook自动注入能力](#为namespace开启webhook自动注入能力)
    - [创建业务Pod](#创建业务pod)
    - [Fuse挂载点自动恢复测试](#fuse挂载点自动恢复测试)
  - [7-Fluid升级过程中使用Dataset](#7-fluid升级过程中使用dataset)
    - [准备工作](#准备工作-2)
    - [升级Fluid](#升级fluid)
    - [新版本下创建Dataset](#新版本下创建dataset)
    - [访问Dataset中数据](#访问dataset中数据)
    - [环境清理](#环境清理-2)
  - [8-设置AlluxioRuntime各组件Pod的资源需求](#8-设置alluxioruntime各组件pod的资源需求)
    - [创建带资源需求的AlluxioRuntime](#创建带资源需求的alluxioruntime)
    - [使用Dataset访问数据](#使用dataset访问数据)
    - [检查Alluxio各组件资源需求](#检查alluxio各组件资源需求)
  - [9-Fuse客户端全局部署模式下扩容AlluxioRuntime](#9-fuse客户端全局部署模式下扩容alluxioruntime)
    - [查看节点信息](#查看节点信息)
    - [创建Dataset](#创建dataset)
    - [创建AlluxioRuntime](#创建alluxioruntime)
    - [查看alluxio-worker所在节点](#查看alluxio-worker所在节点)
    - [创建Nginx容器](#创建nginx容器)
    - [扩容AlluxioRuntime](#扩容alluxioruntime)

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

## 2-使用Dataset访问WebUFS

### 创建Dataset和AlluxioRuntime

**创建Dataset CRD和AlluxioRuntime CRD描述文件**
```shell
$ cat << EOF > dataset.yaml
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: hbase
spec:
  mounts:
    - mountPoint: https://mirrors.bit.edu.cn/apache/hbase/stable/
      name: hbase
---
apiVersion: data.fluid.io/v1alpha1
kind: AlluxioRuntime
metadata:
  name: hbase
spec:
  replicas: 2
  tieredstore:
    levels:
      - mediumtype: MEM
        path: /dev/shm
        quota: 2Gi
        high: "0.95"
        low: "0.7"
EOF
```

**创建Dataset和AlluxioRuntime**
```
$ kubectl create -f dataset.yaml
dataset.data.fluid.io/hbase created
alluxioruntime.data.fluid.io/hbase created
```

### 检查Dataset和AlluxioRuntime状态

**检查Dataset状态**
```
$ kubectl get dataset hbase
NAME    UFS TOTAL SIZE   CACHED   CACHE CAPACITY   CACHED PERCENTAGE   PHASE      AGE
hbase   [Calculating]                                                  NotBound   35s
```
此时Dataset处于未绑定(NotBound)状态，即Alluxio仍然在启动中

一段时间后，Alluxio正常启动，Dataset状态变为已绑定(Bound)状态
```
$ kubectl get dataset hbase
NAME    UFS TOTAL SIZE   CACHED      CACHE CAPACITY   CACHED PERCENTAGE   PHASE   AGE
hbase   566.22MiB        566.22MiB   4.00GiB          100.0%              Bound   6m14s
```

**检查AlluxioRuntime状态**
```
$ kubectl get alluxioruntime hbase
NAME    MASTER PHASE   WORKER PHASE   FUSE PHASE   AGE
hbase   Ready          Ready          Ready        8m3s
```
应当发现Alluxio的各个组件(Alluxio Master, Alluxio Worker, Alluxio Fuse)均为Ready状态

**检查创建的PV,PVC**
```
$ kubectl get pv,pvc
NAME                             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM           STORAGECLASS   REASON   AGE
persistentvolume/default-hbase   100Gi      ROX            Retain           Bound    default/hbase   fluid                   9m13s

NAME                          STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/hbase   Bound    default-hbase   100Gi      ROX            fluid          9m13s
```
应当发现与Dataset相关的PV和PVC被成功创建:
- PV名为"{Dataset命名空间}-{Dataset名}"，例如`default-hbase`
- PVC名为"{Dataset名}",例如`hbase`
- PVC绑定的Volume为"{Dataset命名空间}-{Dataset名}"

**检查AlluxioRuntime对应的Pod是否正常运行**

```
$ kubectl get pod -l release=hbase
NAME             READY   STATUS    RESTARTS   AGE
hbase-master-0   2/2     Running   0          18m
hbase-worker-0   2/2     Running   0          17m
hbase-worker-1   2/2     Running   0          17m
```
应当发现:
- 全部Pod正常运行，处于Running状态，且RESTARTS均为0
- 包含1个Alluxio Master Pod(`hbase-master-0`)
- 包含2各Alluxio Worker Pod(`hbase-worker-X`)
- 不包含Alluxio Fuse Pod

如果以上条件都满足，说明AlluxioRuntime正常运行。

### 通过Dataset访问数据

**创建Nginx Pods**

```
$ cat << EOF > nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: hbase-vol
  volumes:
    - name: hbase-vol
      persistentVolumeClaim:
        claimName: hbase
```

```
$ kubectl create -f nginx.yaml
pod/nginx created
```

**检查Alluxio Fuse Pod状态**
```
$ kubectl get pod -o wide
NAME               READY   STATUS    RESTARTS   AGE   IP            NODE                     NOMINATED NODE   READINESS GATES
hbase-fuse-5qqb4   1/1     Running   0          13m   172.16.0.26   cn-beijing.172.16.0.26   <none>           <none>
hbase-master-0     2/2     Running   0          51m   172.16.0.27   cn-beijing.172.16.0.27   <none>           <none>
hbase-worker-0     2/2     Running   0          50m   172.16.0.27   cn-beijing.172.16.0.27   <none>           <none>
hbase-worker-1     2/2     Running   0          50m   172.16.0.26   cn-beijing.172.16.0.26   <none>           <none>
nginx              1/1     Running   0          13m   10.5.0.14     cn-beijing.172.16.0.26   <none>           <none>
```
应当发现，Alluxio Fuse Pod(`hbase-fuse-xxxxx`)正常运行，并且与Nginx Pod在同一节点


**登录到Nginx Pod容器中并访问数据**
```
$ kubectl exec -it nginx bash

$ ls -lR /data/
/data/:
total 1
dr--r----- 1 root root 6 Feb 14 07:38 hbase

/data/hbase:
total 579807
-r--r----- 1 root root    101357 Dec 18 03:35 CHANGES.md
-r--r----- 1 root root   1058578 Dec 18 03:35 RELEASENOTES.md
-r--r----- 1 root root         0 Dec 18 03:35 api_compare_2.4.8_to_2.4.9RC0.html
-r--r----- 1 root root 283496242 Dec 18 03:35 hbase-2.4.9-bin.tar.gz
-r--r----- 1 root root 272026535 Dec 18 03:35 hbase-2.4.9-client-bin.tar.gz
-r--r----- 1 root root  37038972 Dec 18 03:35 hbase-2.4.9-src.tar.gz

$ time cp -r /data/hbase ./

real    1m30.605s
user    0m0.005s
sys     0m1.177s
```
应当发现:
1. 能够正常访问文件元信息（`ls -lR`）
2. 能够读取文件数据（`cp -r /data/hbase`）

### 环境清理

**删除Nginx Pod**

```
$ kubectl delete -f nginx.yaml
pod "nginx" deleted
```

**检查Alluxio Pod状态**

```
$ kubectl get pod -l release=hbase
NAME               READY   STATUS    RESTARTS   AGE
hbase-fuse-5qqb4   1/1     Running   0          19m
hbase-master-0     2/2     Running   0          57m
hbase-worker-0     2/2     Running   0          56m
hbase-worker-1     2/2     Running   0          56m
```
应当发现Alluxio Fuse Pod没有随Nginx Pod被删除而被删除

**删除Dataset**

```
$ kubectl delete dataset hbase
dataset.data.fluid.io "hbase" deleted
```

由于Dataset被删除，其绑定的AlluxioRuntime也会被级联删除

```
$ kubectl get alluxioruntime
No resources found in default namespace.

$ kubectl get pod -l release=hbase
No resources found in default namespace.
```

一段时间后，应当发现，AlluxioRuntime与Alluxio Pods均被删除


## 3-DataLoad数据预加载

本节展示了如何借助DataLoad进行数据预加载，同样以hbase数据集为例。首先创建好相应的Dataset和AlluxioRuntime。

1. [创建Dataset和AlluxioRuntime](#创建dataset和alluxioruntime)

2. [判断Dataset是否bound](#检查dataset和alluxioruntime状态)

> 需要注意的是，创建DataLoad进行数据预加载并不需要Dataset处于bound的状态，这里多一步判断Dataset是否bound只是可以为后续出错排查提供方便。

**查看Dataset当前缓存的数据量**

```
$ kubectl get dataset hbase
NAME    UFS TOTAL SIZE   CACHED   CACHE CAPACITY   CACHED PERCENTAGE   PHASE   AGE
hbase   566.22MiB        0.00B    4.00GiB          0.0%                Bound   9m41s
```
应当发现，初始时没有缓存任何数据（`Cached=0.00B`）

### 创建DataLoad

```
$ cat << EOF > dataload.yaml
apiVersion: data.fluid.io/v1alpha1
kind: DataLoad
metadata:
  name: hbase-dataload
spec:
  dataset:
    name: hbase
    namespace: default
EOF
```

输入以下命令创建DataLoad。

```
$ kubectl create -f dataload.yaml
dataload.data.fluid.io/hbase-dataload created
```

### 检查DataLoad状态

**检查DataLoad是否已经创建Job**

```
$ kubectl get job | awk '$1=="hbase-dataload-loader-job"'
hbase-dataload-loader-job   0/1           52s        65s
```
应当发现名为`hbase-dataload-loader-job`存在

**检查DataLoad对应的Pod的状态**

```
$ kubectl get pod -l release=hbase-dataload-loader
NAME                              READY   STATUS      RESTARTS   AGE
hbase-dataload-loader-job-rmkxx   0/1     Running 0          1m6s
```

应当发现DataLoad对应的Pod处于`Running`状态。

```
$ kubectl get dataload hbase-dataload
NAME             DATASET   PHASE      AGE     DURATION
hbase-dataload   hbase     Loading    6m32s   
```

应当发现DataLoad处于`Loading`状态。

如果Dataset的size很小的话，加载数据操作会很快完成，完成后DataLoad对应的Pod变为`Completed`状态。

```
$ kubectl get pod -l release=hbase-dataload-loader
NAME                              READY   STATUS      RESTARTS   AGE
hbase-dataload-loader-job-rmkxx   0/1     Completed   0          3m6s
```

```
$ kubectl get dataloads.data.fluid.io
NAME             DATASET   PHASE      AGE     DURATION
hbase-dataload   hbase     Complete   6m32s   57s
```

### 检查DataLoad是否执行成功

检查DataLoad创建的Job的状态，执行成功，输入以下命令返回`Complete`，执行失败，返回`Failed`。如果Job未执行结束，返回空。

```shell
$ kubectl get job hbase-dataload-loader-job -o jsonpath={.status.conditions[0].type}
```

检查DataLoad是否执行成功，如果执行成功，输入以下命令返回`Complete`，如果失败，返回`Failed`。如果DataLoad未结束，输出`Pending`或者`Loading`。

```shell
$ kubectl get dataload | awk '$1=="hbase-dataload" {print $3}'
```

**检查Dataset的数据是否被缓存**

```shell
$ kubectl get dataset hbase
NAME    UFS TOTAL SIZE   CACHED      CACHE CAPACITY   CACHED PERCENTAGE   PHASE   AGE
hbase   566.22MiB        566.22MiB   4.00GiB          100.0%              Bound   30m
```
应当看到，Dataset的缓存占比(`Cached Percentage`)从`0.0%`变为`100.0%`,说明全部数据均已被缓存

### 环境清理

**删除DataLoad**

```
$ kubectl delete dataload hbase-dataload
dataload.data.fluid.io "hbase-dataload" deleted
```

**检查DataLoad对应的Job和Pod**

```
$ kubectl get job
No resources found in default namespace.
```

```
$ kubectl get pod -l release=hbase-dataload-loader
No resources found in default namespace.
```

**删除Dataset**

```
$ kubectl delete dataset hbase
dataset.data.fluid.io "hbase" deleted
```

## 4-DataBackup数据备份

该节展示了如何借助DataBackup对Dataset进行数据备份，同样以spark数据集为例。首先要创建Dataset和对应的AlluxioRuntime，同时确保Dataset已经处于bound状态。

1. [创建Dataset和AlluxioRuntime](#创建dataset和alluxioruntime)

2. [判断Dataset是否bound](#Dataset是否bound)

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



## 5-同一runtime类型条件下不同并发场景使用Patch进行更新节点标签
该部分验证了在同一 runtime 类型条件下不同并发场景可以使用 patch 对节点进行添加标签和删除标签的功能。为了简化并发场景，可以在单个节点上进行实验。
此时需要给特定的节点进行添加标签 fluid=multi-dataset ：
```shell
kubectl label node <nodeName> fluid=multi-dataset
```

### 准备工作
该部分需要创建多个 dataset 和 runtime 的 yaml 文件，并且 dataset 只能够调度到添加标签的节点上。本实验中将 dataset 和 runtime 放在同一个 yaml 文件
中，文件名称分别为 hbase1.yaml、hbase2.yaml、hbase3.yaml、hbase4.yaml、hbase5.yaml。他们对应的 runtime 都是 alluxioruntime。这里需要注意的是本功能仅支持同一类型的 runtime 进行操作，并不支持对于多种类型 runtime 进行同时调度到一个节点的情况。
其中 hbase1.yaml 文件内容如下：
```shell
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: hbase1
spec:
  mounts:
    - mountPoint: https://mirrors.tuna.tsinghua.edu.cn/apache/hbase/stable/
      name: hbase
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: fluid
              operator: In
              values:
                - "multi-dataset"
  placement: "Shared"

---
apiVersion: data.fluid.io/v1alpha1
kind: AlluxioRuntime
metadata:
  name: hbase1
spec:
  replicas: 1
  tieredstore:
    levels:
      - mediumtype: MEM
        path: /dev/shm
        quota: 2Gi
        high: "0.95"
        low: "0.7"
```

### 多数据集并发调度到同一节点
这一部分检验一下在多个 dataset 同时进行调度到某个节点时，该节点中的标签能否进行正确添加。
首先创建脚本 add.sh：
```shell
kubectl apply -f hbase1.yaml
kubectl apply -f hbase2.yaml
kubectl apply -f hbase3.yaml
```
经过一段时间后，可以发现对应的 pod 均创建完毕：
```shell
# kubectl get pods 
NAME                           READY   STATUS    RESTARTS   AGE
hbase1-fuse-lnvvb              1/1     Running   0          4s
hbase1-master-0                2/2     Running   0          40s
hbase1-worker-jbj5b            2/2     Running   0          4s
hbase2-fuse-7lp64              1/1     Running   0          6s
hbase2-master-0                2/2     Running   0          39s
hbase2-worker-cw76l            2/2     Running   0          6s
hbase3-fuse-rphq5              1/1     Running   0          5s
hbase3-master-0                2/2     Running   0          38s
hbase3-worker-zn2cf            2/2     Running   0          5s

```

此时查看对应节点的标签信息，发现对应的标签已经进行了添加：
```shell
Labels:             ...
                    fluid.io/dataset-num=3
                    fluid.io/s-alluxio-default-hbase1=true
                    fluid.io/s-alluxio-default-hbase2=true
                    fluid.io/s-alluxio-default-hbase3=true
                    fluid.io/s-default-hbase1=true
                    fluid.io/s-default-hbase2=true
                    fluid.io/s-default-hbase3=true
                    fluid.io/s-h-alluxio-m-default-hbase1=2GiB
                    fluid.io/s-h-alluxio-m-default-hbase2=2GiB
                    fluid.io/s-h-alluxio-m-default-hbase3=2GiB
                    fluid.io/s-h-alluxio-t-default-hbase1=2GiB
                    fluid.io/s-h-alluxio-t-default-hbase2=2GiB
                    fluid.io/s-h-alluxio-t-default-hbase3=2GiB
                    ...
```

此时可以发现，标签 fluid.io/dataset-num 正确的统计了该节点上的 dataset 的数量，fluid.io/s-h-alluxio-m-default-hbase1、fluid.io/s-h-alluxio-t-default-hbase1 正确统计了 hbase1 的内存占用的大小和总占用空间的大小。
另外 fluid.io/s-alluxio-default-hbase1=true，fluid.io/s-default-hbase1=true 这两个标签表明了 name 为 hbase1 的 dataset 调度在该节点上。其余 dataset 对应标签均类似。这表明了通过 patch 在多 dataset 同时调度到某节点时可以正确给节点添加上标签。

### 多数据集在同一节点并发调度与删除
这一部分检验一下在部分 dataset 进行调度，部分 dataset 进行删除时，节点中的标签能否进行添加或者删除。
首先创建脚本 add-delete.sh：
```shell
kubectl apply -f alluxioruntime4.yaml
kubectl apply -f alluxioruntime5.yaml
kubectl delete dataset hbase1
kubectl delete dataset hbase2
```
经过一段时间后，可以发现对应的 pod 均创建或者删除完毕：
```shell
# kubectl get pods 
NAME                           READY   STATUS    RESTARTS   AGE
hbase3-fuse-rphq5              1/1     Running   0          5m21s
hbase3-master-0                2/2     Running   0          5m54s
hbase3-worker-zn2cf            2/2     Running   0          5m21s
hbase4-fuse-28t2q              1/1     Running   0          4s
hbase4-master-0                2/2     Running   0          36s
hbase4-worker-xcflx            2/2     Running   0          4s
hbase5-fuse-cpz4r              1/1     Running   0          5s
hbase5-master-0                2/2     Running   0          38s
hbase5-worker-fgww4            2/2     Running   0          5s
```
此时查看对应节点的标签信息，发现对应的标签已经进行了添加或者删除：
```shell
Labels:             ...         
                    fluid=multi-dataset
                    fluid.io/dataset-num=3
                    fluid.io/s-alluxio-default-hbase3=true
                    fluid.io/s-alluxio-default-hbase4=true
                    fluid.io/s-alluxio-default-hbase5=true
                    fluid.io/s-default-hbase3=true
                    fluid.io/s-default-hbase4=true
                    fluid.io/s-default-hbase5=true
                    fluid.io/s-h-alluxio-m-default-hbase3=2GiB
                    fluid.io/s-h-alluxio-m-default-hbase4=2GiB
                    fluid.io/s-h-alluxio-m-default-hbase5=2GiB
                    fluid.io/s-h-alluxio-t-default-hbase3=2GiB
                    fluid.io/s-h-alluxio-t-default-hbase4=2GiB
                    fluid.io/s-h-alluxio-t-default-hbase5=2GiB
                    ...
```
通过节点的标签可以看出，有关 name 为 hbase1 和 hbase2 相关的标签全部都被删除，新增了 name 为 hbase4 和 hbase5 的相关标签。可以发现部分 dataset 进行调度，部分 dataset 进行删除时，可以通过 patch 正确的对标签进行相应处理。

### 多数据集在同一节点并发删除
这一部分检验一下多个 dataset 同时进行删除时，节点中的标签能否进行删除。
首先创建脚本 delete.sh：
```shell
kubectl delete dataset --all
```
经过一段时间后，可以发现对应的 pod 均删除完毕：
```shell
# kubectl get pods 
No resources found in default namespace.
```
此时查看对应节点的标签信息，发现对应的标签已经进行了删除：
```shell
Labels:             ...
                    fluid=multi-dataset
                    ...
```
通过节点的标签可以发现该节点上之前添加的 dataset 相关标签全部被删除了，从而表明了 patch 操作对于多个 dataset 同时删除操作过程中操作标签的正确性。

这里需要注意的是目前该功能只支持同种类型的 runtime 进行并发的进行调度和删除，并不支持对于多种类型 runtime 进行同时调度和删除。

## 6-Fuse挂载点自动恢复

### 准备工作

正常安装Fluid v0.7.0+版本，并启用Fuse挂载点自动恢复功能:
```shell
$ helm install --set csi.recoverFusePeriod=30 fluid charts/fluid/fluid
```

Fluid所有Pod均正常运行:
```
$ kubectl get pod -n fluid-system
NAME                                        READY   STATUS    RESTARTS   AGE
alluxioruntime-controller-ddbb764fb-v5x6j   1/1     Running   0          91m
csi-nodeplugin-fluid-hpmfx                  2/2     Running   0          91m
dataset-controller-57477b5fcd-hhgkk         1/1     Running   0          91m
fluid-webhook-6c457896dc-dhx6f              1/1     Running   0          91m
jindoruntime-controller-676755d897-fjd5w    1/1     Running   0          91m
```

查看Fluid CSI Plugin日志, 确认Fuse挂载点自动恢复功能正常启动，看到如下日志:
```
$ kubectl logs -n fluid-system csi-nodeplugin-fluid-hpmfx -c plugins
...
I0121 19:03:58.920429  863993 recover.go:70] start csi recover
...
```

### 创建Dataset和AlluxioRuntime

```yaml
cat << EOF > dataset.yaml
kind: Dataset
metadata:
  name: hbase
spec:
  mounts:
    - mountPoint: https://mirrors.tuna.tsinghua.edu.cn/apache/hbase/stable
      name: hbase
---
apiVersion: data.fluid.io/v1alpha1
kind: AlluxioRuntime
metadata:
  name: hbase
spec:
  replicas: 1
  tieredstore:
    levels:
      - mediumtype: MEM
        path: /dev/shm
        quota: 2Gi
        high: "0.95"
        low: "0.7"
EOF
```

```shell
$ kubectl create -f dataset.yaml
```

等待Dataset和AlluxioRuntime正常启动:
```shell
$ kubectl get dataset,alluxioruntime
NAME                          UFS TOTAL SIZE   CACHED   CACHE CAPACITY   CACHED PERCENTAGE   PHASE   AGE
dataset.data.fluid.io/hbase   566.22MiB        0.00B    2.00GiB          0.0%                Bound   51s

NAME                                 MASTER PHASE   WORKER PHASE   FUSE PHASE   AGE
alluxioruntime.data.fluid.io/hbase   Ready          Ready          Ready        51s
```

### 为Namespace开启Webhook自动注入能力

Fuse挂载点自动恢复功能需要确保pod的mountPropagation设置为`HostToContainer`或`Bidirectional`。Fluid Webhook能够检查并注入mountPropagation字段，以保证Fuse挂载点自动恢复功能正常运作。具体信息请参考[此处](https://github.com/fluid-cloudnative/fluid/blob/master/docs/zh/samples/fuse_recover.md)

为`default` namespace开启Webhook注入功能，开启后，所有`default` namepsace下挂载Fluid Dataset PVC的Pod都会被注入上述信息:
```shell
$ kubectl patch ns default -p '{"metadata": {"labels": {"fluid.io/enable-injection": "true"}}}'

$ kubectl get ns default --show-labels
NAME      STATUS   AGE     LABELS
default   Active   4d12h   fluid.io/enable-injection=true
```

### 创建业务Pod

创建Nginx Pod模拟Fluid Dataset的数据消费者:
```yaml
$ cat << EOF > nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: hbase-vol
          # mountPropagation: HostToContainer (auto-injected by webhook)
  volumes:
    - name: hbase-vol
      persistentVolumeClaim:
        claimName: hbase
EOF
```

```shell
$ kubectl create -f nginx.yaml
```

查看Webhook信息注入是否正常:
```shell
$ kubectl get pod nginx -o yaml | grep -B1 -A1 mountPropagation
...
      - mountPath: /data
        mountPropagation: HostToContainer
        name: hbase-vol
```

### Fuse挂载点自动恢复测试

当Nginx Pod正常启动后，可在Fuse挂载点处正常访问数据集中的文件:
```shell
$ kubectl exec -it nginx -- ls /data/hbase
CHANGES.md                          hbase-2.4.9-bin.tar.gz
RELEASENOTES.md                     hbase-2.4.9-client-bin.tar.gz
api_compare_2.4.8_to_2.4.9RC0.html  hbase-2.4.9-src.tar.gz
```

此时，删除Alluxio Fuse Pod，模拟Alluxio Fuse异常崩溃情况:
```shell
$ kubectl delete pod hbase-fuse-gcjsc
```

检查Nginx Pod中的数据访问，应当发现'Transport endpoint is not connected'问题
```shell
kubectl exec -it nginx -- ls /data/hbase
ls: cannot open directory '/data/hbase': Transport endpoint is not connected
```

等待一段时间(约30s), 观测到Fluid Dataset上报出的Fuse挂载点已自动恢复的事件后:
```shell
Normal  FuseRecoverSucceed  75s (x1 over 2m15s)  FuseRecover  Fuse recover /var/lib/kubelet/pods/5fbfc0f0-9a1e-41fe-9d3f-5cca3daa24a8/volumes/kubernetes.io~csi/default-hbase/mount succeed
```

再次检查Nginx Pod中的数据访问，此时恢复正常的数据访问能力:
```shell
kubectl exec -it nginx -- ls /data/hbase
CHANGES.md                          hbase-2.4.9-bin.tar.gz
RELEASENOTES.md                     hbase-2.4.9-client-bin.tar.gz
api_compare_2.4.8_to_2.4.9RC0.html  hbase-2.4.9-src.tar.gz
```

## 7-Fluid升级过程中使用Dataset

以下例子以Fluid v0.6升级至0.7为例，测试Fluid升级对使用Dataset过程是否有影响。

### 准备工作

**安装旧版本Fluid**

```
$ git clone https://github.com/fluid-cloudnative/fluid.git /fluid
```

```
$ kubectl create ns fluid-system

$ helm install fluid /fluid/charts/fluid/v0.6.0
```

**旧版本下使用Dataset**

```
$ cat << EOF > old_dataset.yaml
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: hbase
spec:
  mounts:
    - mountPoint: https://mirrors.bit.edu.cn/apache/hbase/stable/
      name: hbase
---
apiVersion: data.fluid.io/v1alpha1
kind: AlluxioRuntime
metadata:
  name: hbase
spec:
  replicas: 2
  tieredstore:
    levels:
      - mediumtype: MEM
        path: /dev/shm
        quota: 2Gi
        high: "0.95"
        low: "0.7"
EOF
```

```
$ kubectl create -f old_dataset.yaml
dataset.data.fluid.io/hbase created
alluxioruntime.data.fluid.io/hbase created
```

检查Alluxio Pod状态:
```
$ kubectl get pod -l release=hbase
NAME                 READY   STATUS              RESTARTS   AGE
hbase-fuse-6fkrj     0/1     ContainerCreating   0          38s
hbase-fuse-rwbr6     1/1     Running             0          39s
hbase-master-0       2/2     Running             0          67s
hbase-worker-dxxzl   0/2     ContainerCreating   0          38s
hbase-worker-tj8xb   2/2     Running             0          39s
```

### 升级Fluid

**使用Helm升级Fluid**

```
$ helm upgrade fluid /fluid/charts/fluid/fluid
```

**升级Fluid CRDs**
```
$ kubectl apply -f /fluid/charts/fluid/fluid/crds
```

### 新版本下创建Dataset

**创建Dataset**

在不删除旧版本下创建的Dataset的同时，创建一个新的Dataset，使两者共存

```
$ cat << EOF > new_dataset.yaml
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: another-hbase
spec:
  mounts:
    - mountPoint: https://mirrors.bit.edu.cn/apache/hbase/stable/
      name: hbase
---
apiVersion: data.fluid.io/v1alpha1
kind: AlluxioRuntime
metadata:
  name: another-hbase
spec:
  replicas: 2
  tieredstore:
    levels:
      - mediumtype: MEM
        path: /dev/shm
        quota: 2Gi
        high: "0.95"
        low: "0.7"
EOF
```

```
$ kubectl create -f new_dataset.yaml
dataset.data.fluid.io/another-hbase created
alluxioruntime.data.fluid.io/another-hbase created
```

### 访问Dataset中数据

**访问旧版本Dataset中数据**

```
$ cat << EOF > old_nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: hbase-vol
  volumes:
    - name: hbase-vol
      persistentVolumeClaim:
        claimName: hbase
EOF
```

```
$ kubectl create -f old_nginx.yaml
```

登录进入创建的Nginx Pod，应当发现能够正常访问旧版本Dataset中的数据

```
$ kubectl exec -it nginx bash

$ ls -lR /data/

$ cp -r /data/ ./
```

**访问新版本Dataset中数据**

```
$ cat << EOF > new_nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: another-nginx
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: hbase-vol
  volumes:
    - name: hbase-vol
      persistentVolumeClaim:
        claimName: another-hbase
EOF
```

```
$ kubectl create -f new_nginx.yaml
pod/another-nginx created
```

登录进入创建的Nginx Pod，应当发现能够正常访问新版本Dataset中的数据

```
$ kubectl exec -it another-nginx bash

$ ls -lR /data

$ cp -r /data ./
```

### 环境清理

**删除旧版本Dataset**

```
$ kubectl delete pod nginx

$ kubectl delete dataset hbase
```
应当发现，Nginx Pod和旧版本Dataset均正常删除，且绑定的AlluxioRuntime也被级联删除。

**删除新版本Dataset**
```
$ kubectl delete pod another-nginx

$ kubectl delete dataset another-hbase
```

应当发现，Nginx Pod和新版本Dataset均正常删除，且绑定的AlluxioRuntime也被级联删除。

## 8-设置AlluxioRuntime各组件Pod的资源需求

### 创建带资源需求的AlluxioRuntime

```
$ cat << EOF > dataset.yaml
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: hbase
spec:
  mounts:
    - mountPoint: https://mirrors.bit.edu.cn/apache/hbase/stable/
      name: hbase
---
apiVersion: data.fluid.io/v1alpha1
kind: AlluxioRuntime
metadata:
  name: hbase
spec:
  replicas: 1
  tieredstore:
    levels:
      - mediumtype: MEM
        path: /dev/shm
        quota: 2Gi
        high: "0.95"
        low: "0.7"
  master:
    resources:
      requests:
        cpu: 1000m
        memory: 4Gi
      limits:
        cpu: 2000m
        memory: 8Gi
  jobMaster:
    resources:
      requests:
        cpu: 1500m
        memory: 4Gi
      limits:
        cpu: 2000m
        memory: 8Gi
  worker:
    resources:
      requests:
        cpu: 1000m
        memory: 4Gi
      limits:
        cpu: 2000m
        memory: 8Gi
  jobWorker:
    resources:
      requests:
        cpu: 1000m
        memory: 4Gi
      limits:
        cpu: 2000m
        memory: 8Gi
  fuse:
    resources:
      requests:
        cpu: 1000m
        memory: 4Gi
      limits:
        cpu: 2000m
        memory: 8Gi
EOF
```

```
$ kubectl create -f dataset.yaml
dataset.data.fluid.io/hbase created
alluxioruntime.data.fluid.io/hbase created
```

### 使用Dataset访问数据

创建使用该Dataset的Pod，使Alluxio Fuse Pod被启动
```
$ cat << EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: hbase-vol
  volumes:
    - name: hbase-vol
      persistentVolumeClaim:
        claimName: hbase
```

```
$ kubectl create -f pod.yaml
```

### 检查Alluxio各组件资源需求

**Alluxio Master Pod**

```
$ kubectl get pod hbase-master-0 -o=jsonpath='{@.spec.containers[*].resources}' | jq
{
  "limits": {
    "cpu": "2",
    "memory": "8Gi"
  },
  "requests": {
    "cpu": "1",
    "memory": "4Gi"
  }
}
{
  "limits": {
    "cpu": "2",
    "memory": "8Gi"
  },
  "requests": {
    "cpu": "1500m",
    "memory": "4Gi"
  }
}
```

**Alluxio Worker Pod**
```
$ kubectl get pod hbase-worker-0 -o=jsonpath='{@.spec.containers[*].resources}' | jq
{
  "limits": {
    "cpu": "2",
    "memory": "10Gi"
  },
  "requests": {
    "cpu": "1",
    "memory": "4Gi"
  }
}
{
  "limits": {
    "cpu": "2",
    "memory": "8Gi"
  },
  "requests": {
    "cpu": "1",
    "memory": "4Gi"
  }
}
```

**Alluxio Fuse Pod**

```
$ kubectl get pod hbase-fuse-4stx5 -o=jsonpath='{@.spec.containers[*].resources}' | jq
{
  "limits": {
    "cpu": "2",
    "memory": "10Gi"
  },
  "requests": {
    "cpu": "1",
    "memory": "4Gi"
  }
}
```

应当发现Alluxio各组件Pod的资源需求均正确设置。


## 9-Fuse客户端全局部署模式下扩容AlluxioRuntime


### 查看节点信息

查看集群中有哪些节点：

```shell
kubectl get nodes
```

本测试用例需要至少5个节点才能完成测试。下面将分别称呼它们为node1~node5。

为node1~node4打上标签：

```shell
kubectl label node <nodeName> fuse=true
```

### 创建Dataset

检查待创建的Dataset对象：

```shell
$ cat<<EOF >dataset.yaml
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: hbase
spec:
  mounts:
    - mountPoint: https://mirrors.tuna.tsinghua.edu.cn/apache/hbase/stable/
      name: hbase
EOF
```

创建Dataset：

```shell
kubectl create -f dataset.yaml
```

### 创建AlluxioRuntime

检查待创建的AlluxioRuntime对象：

```
cat<<EOF >runtime-node-selector.yaml
apiVersion: data.fluid.io/v1alpha1
kind: AlluxioRuntime
metadata:
  name: hbase
spec:
  replicas: 1
  tieredstore:
    levels:
      - mediumtype: MEM
        path: /dev/shm
        quota: 2Gi
        high: "0.95"
        low: "0.7"
  properties:
    alluxio.user.block.size.bytes.default: 256MB
    alluxio.user.streaming.reader.chunk.size.bytes: 256MB
    alluxio.user.local.reader.chunk.size.bytes: 256MB
    alluxio.worker.network.reader.buffer.size: 256MB
    alluxio.user.streaming.data.timeout: 300sec
  fuse:
    global: true
    nodeSelector:
      fuse: true
    args:
      - fuse
      - --fuse-opts=kernel_cache,ro,max_read=131072,attr_timeout=7200,entry_timeout=7200,nonempty,max_readahead=0
EOF
```
该配置文件中，fuse设置为了全局模式，fuse客户端会运行在刚刚打了标签的node1~node4上。

创建AlluxioRuntime：

```shell
kubectl create -f runtime.yaml
```

### 查看alluxio-worker所在节点

查看该Dataset对应的alluxio-worker所在节点：

```shell
kubectl get pod -o custom-columns=NAME:metadata.name,NAME:.spec.nodeName | grep hbase-worker
```

目前应该只有1副本，假设它所在的节点是node1。

### 创建Nginx容器

检查待创建的Pod对象：

```shell
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: hbasevol
  nodeSelector:
    kubernetes.io/hostname: <nodeName>
  volumes:
    - name: hbasevol
      persistentVolumeClaim:
        claimName: hbase
```

这样的Pod需要创建3个，前两个的hostname指定为node2，第三个的hostname指定为node3。

这样，node2~4上将分别有2、1、0个正在使用该数据集的Pod。

### 扩容AlluxioRuntime

修改AlluxioRuntime的.spec.replicas，依次修改为2~5。

每次修改后，都再次查看该Dataset对应的alluxio-worker所在节点：

```shell
kubectl get pod -o custom-columns=NAME:metadata.name,NAME:.spec.nodeName | grep hbase-worker
```

可以看到，新建的alluxio-worker将依次被调度到node2~5。
