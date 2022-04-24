---
title: "构建"
linkTitle: "构建"
weight: 40
date: 2022-03-10
description: >
  构建各个项目，将开发需要的依赖等都准备号
---



### kit

在终端中执行以下命令：

```bash
make go.mod
make test
make lint
make check-diff
```



### components-contrib

在终端中执行以下命令：

```bash
make go.mod
make modtidy-all
make test
make lint
make check-diff
```

备注： conf-tests 和 e2e-tests-zeebe 在本地是跑不起来的。



### dapr

在终端中执行以下命令：

```bash
make go.mod
make modtidy-all
make test
make lint
make check-diff
```

备注： conf-tests 和 e2e-tests-zeebe 在本地是跑不起来的。

