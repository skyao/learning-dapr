---
title: "Fork Dapr相关的仓库"
linkTitle: "Fork仓库"
weight: 10
date: 2022-01-24
description: >
  Fork Dapr相关的仓库，以便修改和提交PR
---

### fork 仓库

将 https://github.com/dapr/ 组织下的各个需要用到的仓库都 fork 一遍，如：

- https://github.com/skyao/dapr
- https://github.com/skyao/components-contrib
- https://github.com/skyao/kit
- https://github.com/skyao/go-sdk/
- https://github.com/skyao/java-sdk/

以及一些部分同学可能不需要关注的仓库：

- https://github.com/skyao/docs/
- https://github.com/skyao/quickstarts

如果已经 fork 了，注意 master 分支到最新(GitHub 页面上点 "Fetch upstream" 即可)。

注意：绝对不要在 master 分支上直接修改代码。

### clone 社区仓库到本地

```bash
mkdir -p ~/work/code/dapr
cd ~/work/code/dapr

git clone git@github.com:dapr/dapr.git
git clone git@github.com:dapr/components-contrib.git
git clone git@github.com:dapr/kit.git
git clone git@github.com:dapr/quickstarts.git

git clone git@github.com:dapr/java-sdk.git
git clone git@github.com:dapr/go-sdk.git
```

### clone fork 仓库到本地

```bash
mkdir -p ~/work/code/dapr-fork
cd ~/work/code/dapr-fork

git clone git@github.com:skyao/dapr.git
git clone git@github.com:skyao/components-contrib.git
git clone git@github.com:skyao/kit.git
git clone git@github.com:skyao/quickstarts.git

git clone git@github.com:skyao/java-sdk.git
git clone git@github.com:skyao/go-sdk.git
```