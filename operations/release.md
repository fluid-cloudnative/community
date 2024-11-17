---

# Release Process

## Overview

1. Update "What's New"
2. Create a tag and release
3. Publish the release, write the Release Note, and upload the package
4. Update the Charts project
5. Update the master branch to the new version

## Detailed Steps

### 1. Update "What's New"

1. Fork the project to your own repository on GitHub.

2. Checkout a new branch from your repository:

   ```bash
   git clone https://github.com/{your-repo}/fluid.git
   cd fluid
   git checkout -b update_readme_release
   ```

3. Update the document: edit "What is NEW!" in `$FLUID_HOME/ReadMe.md` (both CN and EN), optionally add a video link.

4. Commit the changes:

   ```bash
   git add --all
   git commit -s -m "Update what's new for 0.5.0"
   ```

5. Create a Pull Request, request a review, and proceed to the next step after merging.

### 2. Create a Tag (Carefully perform this in the project repo)

1. Clone the project:

   ```bash
   git clone https://github.com/fluid-cloudnative/fluid.git fluid-master
   cd fluid-master
   ```

2. Ensure the code is up-to-date:

   ```bash
   git pull
   ```

3. Create a new version tag, e.g., 0.5.0:

   ```bash
   git tag v0.5.0
   git push origin v0.5.0
   ```

4. Generate the Fluid package:

   ```bash
   cd charts/fluid
   helm package fluid
   # Output: Successfully packaged chart and saved it to: fluid-master/charts/fluid/fluid-0.5.0.tgz
   ```

5. Create a branch, e.g., v0.5:

   ```bash
   git checkout -b v0.5
   git push origin v0.5
   ```

### 3. Publish Release, Write Release Note, Upload Package

1. Log in to GitHub and visit the [Releases page](https://github.com/fluid-cloudnative/fluid/releases), click `Draft a new release`.

2. Select v0.5.0 and fill in the following content:

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

3. Upload the package file.

### 4. Publish Helm Chart

1. Create a branch for charts and update:

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

   echo $? # Ensure the return value is 0
   ```

2. Create historical version:

   ```bash
   bash create_history_version.sh

   git add --all
   git commit -s -m "Update helm-chart-fluid-0.5.0"
   git push
   ```

### 5. Update Master Branch Version

1. Change the line containing `VERSION=` in the Makefile to the new version 0.6.0.

2. Update the `version:` line in `charts/fluid/fluid/Chart.yaml` to 0.6.0.

---