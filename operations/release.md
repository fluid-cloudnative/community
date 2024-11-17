# 发布流程

## 流程概述

1. 更新what’s new  
2. 创建tag，并发布
3. 发布Release，并且编写Release Note, 上传package
4. 更新Charts项目
5. 在 master 分支中将已发布的版本更新为新版本


## 操作细节


### 1. 更新what’s new 

1. 在 GitHub 页面 Fork 项目到自己的仓库。

2. 从自己的仓库中 checkout 出新的分支：

```
git clone https://github.com/{your-repo}/fluid.git
cd fluid
git checkout -b update_readme_release
```

3. 更新文档

编辑 $FLUID_HOME/ReadMe.md 中的 "What is NEW!"（包括中文和英文），可选添加视频链接。

4. 提交代码

```
git add --all
git commit -s -m "update what's new for 0.5.0"
```

5. 创建Pull Request, 并且请求review, 等待合并完成后进入下一个步骤


### 2. 创建tag （请在项目仓库中小心操作）

1. 从 https://github.com/fluid-cloudnative/fluid.git 克隆项目

```
git clone https://github.com/fluid-cloudnative/fluid.git fluid-master
cd fluid-master
```

2. 确认当前代码为最新版本

```
git log
```

3. 创建新版本 tag, 比如版本为0.5.0 时

```
git tag v0.5.0
git push origin v0.5.0
```

4. 生成fluid package

```
cd fluid-master/charts/fluid
helm package fluid
# 输出： Successfully packaged chart and saved it to: fluid-master/charts/fluid/fluid-0.5.0.tgz
```

5. 创建分支, 比如对应分支为v0.5

```
git checkout -b v0.5
git push v0.5
```

> 下载地址：https://github.com/helm/helm/releases/tag/v3.5.3


### 3. 发布Release，并且编写Release Note, 上传package


1. 登录https://github.com/fluid-cloudnative/fluid/releases 点击 `Draft a new release`

选择v0.5.0, 并且填写markdown

```
## v0.5.0

### Features

- Add Scale out/in support
- Add Metadata Backup and Restore
- Support Fuse global mode, and toleration
- Enhance Prometheous support to Alluxio Runtime
- Support New Runtime： Jindo
- Support HDFS configuration

### Bugs

- [Fix compatibality issue of K8s 1.19+](https://github.com/fluid-cloudnative/fluid/issues/603)

Please check [docs](https://github.com/fluid-cloudnative/fluid/blob/master/docs/zh/TOC.md) to learn how to use Fluid.

**Compatible Alluxio Version**:
Commit: https://github.com/Alluxio/alluxio/commit/42a0cf7df85be3225d226a36b37908d04e8cb595
Branch: https://github.com/Alluxio/alluxio/commits/branch-2.3-fuse
```

2. 上传package包


### 4.  发布Helm Chart到https://github.com/fluid-cloudnative/charts/releases


创建charts的branch, 更新

```
mkdir fluid-cloudnative
cd fluid-cloudnative
git clone https://github.com/fluid-cloudnative/fluid.git
cd fluid
git checkout v0.5.0

cd ../
git clone https://github.com/{your name}/charts.git
cd charts
# 注意是0.5.0，而不是v0.5.0
git checkout -b helm-chart-fluid-0.5.0
bash update-fluid-charts.sh

echo $? #注意返回值是0
0

bash create_history_version.sh

git add --all
git commit -s -m "Update helm-chart-fluid-0.5.0"
git push
```

### 5. 在master branch中将已经发布的版本修改为新版本，比如新版本为0.6.0

1. 将MakeFile中的包含VERSION=的一行的值改为新版本0.6.0

2. 将charts/fluid/fluid/Chart.yaml中version:的一行的值改为0.6.0
