# 发布流程

## 流程概述

1. 更新what’s new  
2. 创建tag，并发布
3. 发布Release，并且编写Release Note, 上传package


## 操作细节


### 1. 更新what’s new 

1. 在github页面Fork 项目到自己的repo

2. 从自己的repo里checkout出新的branch

```
git clone https://github.com/{your repo}/fluid.git
cd fluid
git checkout -b update_readme_release
```

3. 更新文档


4. 提交代码

```
git add --all
git commit -s -m "update what's new for 0.5.0"
```

5. 创建Pull Request, 并且请求review, 等待合并完成后进入下一个步骤


### 2. 创建tag (此操作是在项目repo，因此请小心操作)

1. 从https://github.com/fluid-cloudnative/fluid.git

```
git clone https://github.com/fluid-cloudnative/fluid.git fluid-master
cd fluid-master
```

2. 确认当前代码为最新版本

```
git log
```

3. 创建新版本, 比如版本为0.5.0

```
git tag v0.5.0
git push origin v0.5.0
```

4. 生成fluid package

```
cd fluid-master/charts/fluid
helm package fluid
Successfully packaged chart and saved it to: fluid-master/charts/fluid/fluid-0.5.0.tgz
```

> 下载地址：https://github.com/helm/helm/releases/tag/v3.5.3


### 3. 发布Release，并且编写Release Note, 上传package


1. 登录https://github.com/fluid-cloudnative/fluid/releases， 点击 `Draft a new release`

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
