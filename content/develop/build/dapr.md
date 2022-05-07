---
title: "dapr项目的构建"
linkTitle: "dapr"
weight: 30
date: 2022-03-10
description: >
  dapr/dapr项目存放的是dapr runtime和dapr控制平面的代码
---





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

