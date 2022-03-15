---
title: "Makefile tests部分"
linkTitle: "Makefile tests部分"
weight: 40
date: 2022-03-11
description: >
  和测试相关的make target
---

https://github.com/dapr/dapr/blob/master/Makefile

```makefile
################################################################################
# Target: tests                                                                #
################################################################################
include tests/dapr_tests.mk
```

https://github.com/dapr/dapr/blob/master/tests/dapr_tests.mk



### Target: get-components-contrib

```makefile
################################################################################
# Target: get-components-contrib                                               #
################################################################################
.PHONY: get-components-contrib
get-components-contrib:
	go get github.com/dapr/components-contrib@master
```

获取 components-contrib 仓库 master 分支的最新代码。

执行结果输出如下：

```bash
$ make get-components-contrib 
go get github.com/dapr/components-contrib@master
go: downloading github.com/dapr/components-contrib v1.6.0-rc.1.0.20220310012151-027204f2d3c5
go get: upgraded github.com/dapr/components-contrib v1.6.0-rc.1.0.20220307041340-f1209fb068c7 => v1.6.0-rc.1.0.20220310012151-027204f2d3c5
```



