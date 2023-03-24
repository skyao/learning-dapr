---
title: "components-contrib项目的构建"
linkTitle: "components-contrib"
weight: 20
date: 2022-03-10
description: >
  dapr/components-contrib项目存放的各种组件的代码
---



### components-contrib

在终端中执行以下命令：

```bash
make go.mod
make modtidy-all
make test
make lint
make check-mod-diff
```

备注： conf-tests 和 e2e-tests-zeebe 在本地是跑不起来的。

