---

# 发布流程

## 流程概述

1. 更新 "What's New"
2. 创建 tag 并发布
3. 发布 Release，编写 Release Note，上传 package
4. 更新 Charts 项目
5. 在 master 分支中将已发布的版本更新为新版本

## 操作细节

### 1. 更新 "What's New"

1. 在 GitHub 页面 Fork 项目到自己的仓库。

2. 从自己的仓库中 checkout 出新的分支：

   ```bash
   git clone https://github.com/{your-repo}/fluid.git
   cd fluid
   git checkout -b update_readme_release
   ```

3. 更新文档：编辑 `$FLUID_HOME/ReadMe.md` 中的 "What is NEW!"（包括中文和英文），可选添加视频链接。

4. 提交更改：

   ```bash
   git add --all
   git commit -s -m "Update what's new for 0.5.0"
   ```

5. 创建 Pull Request，请求审核，等待合并完成后进入下一步。

### 2. 创建 tag （请在项目仓库中小心操作）

1. 克隆项目：

   ```bash
   git clone https://github.com/fluid-cloudnative/fluid.git fluid-master
   cd fluid-master
   ```

2. 确认当前代码为最新版本：

   ```bash
   git pull
   ```

3. 创建新版本 tag，例如 0.5.0：

   ```bash
   git tag v0.5.0
   git push origin v0.5.0
   ```

4. 生成 Fluid package：

   ```bash
   cd charts/fluid
   helm package fluid
   # 输出：Successfully packaged chart and saved it to: fluid-master/charts/fluid/fluid-0.5.0.tgz
   ```

5. 创建分支，例如 v0.5：

   ```bash
   git checkout -b v0.5
   git push origin v0.5
   ```

### 3. 发布 Release，编写 Release Note，上传 package

1. 登录 GitHub，访问 [Releases 页面](https://github.com/fluid-cloudnative/fluid/releases)，点击 `Draft a new release`。

2. 选择 v0.5.0，并填写以下内容：

   ```markdown
   ## v0.5.0

   ### Features

   - Add Scale out/in support
   - Add Metadata Backup and Restore
   - Support Fuse global mode, and toleration
   - Enhance Prometheus support to Alluxio Runtime
   - Support New Runtime: Jindo
   - Support HDFS configuration

   ### Bugs

   - [Fix compatibility issue of K8s 1.19+](https://github.com/fluid-cloudnative/fluid/issues/603)

   Please check [docs](https://github.com/fluid-cloudnative/fluid/blob/master/docs/zh/TOC.md) to learn how to use Fluid.

   **Compatible Alluxio Version**:
   - Commit: https://github.com/Alluxio/alluxio/commit/42a0cf7df85be3225d226a36b37908d04e8cb595
   - Branch: https://github.com/Alluxio/alluxio/commits/branch-2.3-fuse
   ```

3. 上传 package 文件。

### 4. 发布 Helm Chart

1. 创建 charts 的分支并更新：

   ```bash
   mkdir fluid-cloudnative
   cd fluid-cloudnative
   git clone https://github.com/fluid-cloudnative/fluid.git
   cd fluid
   git checkout v0.5.0

   cd ../
   git clone https://github.com/{your-name}/charts.git
   cd charts
   git checkout -b helm-chart-fluid-0.5.0
   bash update-fluid-charts.sh

   echo $? # 确保返回值为 0
   ```

2. 创建历史版本：

   ```bash
   bash create_history_version.sh

   git add --all
   git commit -s -m "Update helm-chart-fluid-0.5.0"
   git push
   ```

### 5. 更新 master 分支版本

1. 修改 Makefile 中 `VERSION=` 的值为新版本 0.6.0。

2. 修改 `charts/fluid/fluid/Chart.yaml` 中 `version:` 的值为 0.6.0。

---