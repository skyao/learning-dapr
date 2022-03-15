---
title: "Makefile文件"
linkTitle: "Makefile"
weight: 10
date: 2022-01-21
description: >
  定义和获取变量，定义 test/lint/go.mod/check-diff等target
---

https://github.com/dapr/components-contrib/blob/master/Makefile

## 变量定义

和 kit 仓库完全一致，跳过。

## build target

然后就是正式的 build target 定义了。

其中 target test / target lint / target modtidy 和 kit 仓库完全一致，跳过。

### Target: modtidy-all  

在 modtidy 之外，还提供了一个 modtidy-all 的target，这是因为 conponents-contrib 仓库除了根目录下的 go.mod 之外，还在 `./tests/certification/` 目录下有多个 go.mod 文件。

```bash
find . -name go.mod
./go.mod
./tests/certification/bindings/azure/blobstorage/go.mod
./tests/certification/bindings/azure/cosmosdb/go.mod
./tests/certification/bindings/azure/servicebusqueues/go.mod
./tests/certification/go.mod
./tests/certification/pubsub/rabbitmq/go.mod
./tests/certification/pubsub/mqtt/go.mod
./tests/certification/pubsub/kafka/go.mod
./tests/certification/secretstores/azure/keyvault/go.mod
./tests/certification/state/sqlserver/go.mod
```

```makefile
################################################################################
# Target: modtidy-all                                                          #
################################################################################
MODFILES := $(shell find . -name go.mod)  # 列出所有的 go.mod 文件，列表如上面所示

define modtidy-target
.PHONY: modtidy-$(1)
modtidy-$(1):
	# 进入每一个 go.mod 所在的目录，执行 go mod tidy，然后再返回原来的目录
	cd $(shell dirname $(1)); go mod tidy -compat=1.17; cd -
endef

# Generate modtidy target action for each go.mod file
# 为每个go.mod文件生成modtidy target动作
$(foreach MODFILE,$(MODFILES),$(eval $(call modtidy-target,$(MODFILE))))

# Enumerate all generated modtidy targets
# Note that the order of execution matters: root and tests/certification go.mod
# are dependencies in each certification test. This order is preserved by the
# tree walk when finding the go.mod files.
TIDY_MODFILES:=$(foreach ITEM,$(MODFILES),modtidy-$(ITEM))

# Define modtidy-all action trigger to run make on all generated modtidy targets
.PHONY: modtidy-all
modtidy-all: $(TIDY_MODFILES)
```

实际执行的日志输出如下：

```bash
make modtidy-all
cd .; go mod tidy -compat=1.17; cd -
/home/sky/work/code/dapr/components-contrib
cd ./tests/certification/bindings/azure/blobstorage; go mod tidy -compat=1.17; cd -
/home/sky/work/code/dapr/components-contrib
cd ./tests/certification/bindings/azure/cosmosdb; go mod tidy -compat=1.17; cd -
/home/sky/work/code/dapr/components-contrib
cd ./tests/certification/bindings/azure/servicebusqueues; go mod tidy -compat=1.17; cd -
/home/sky/work/code/dapr/components-contrib
cd ./tests/certification; go mod tidy -compat=1.17; cd -
/home/sky/work/code/dapr/components-contrib
cd ./tests/certification/pubsub/rabbitmq; go mod tidy -compat=1.17; cd -
/home/sky/work/code/dapr/components-contrib
cd ./tests/certification/pubsub/mqtt; go mod tidy -compat=1.17; cd -
/home/sky/work/code/dapr/components-contrib
cd ./tests/certification/pubsub/kafka; go mod tidy -compat=1.17; cd -
/home/sky/work/code/dapr/components-contrib
cd ./tests/certification/secretstores/azure/keyvault; go mod tidy -compat=1.17; cd -
/home/sky/work/code/dapr/components-contrib
cd ./tests/certification/state/sqlserver; go mod tidy -compat=1.17; cd -
/home/sky/work/code/dapr/components-contrib
```

### Target: check-diff 

```makefile
################################################################################
# Target: check-diff                                                           #
################################################################################
.PHONY: check-diff
check-diff:
	.PHONY: check-diff
check-diff:
	git diff --exit-code -- '*go.mod' # check no changes
	git diff --exit-code -- '*go.sum' # check no changes
```

和 kit 仓库的 check-diff 仅仅检查 go.mod 文件不同，component-contrib 的 check-diff 还会检查 go.sum。

> 疑问： 为什么 kit 仓库的 check-diff 不检查 go.sum 文件？


### Target: conf-tests 

执行一致性测试, 其中一致性的测试案例被标注有:

```go
//go:build conftests
// +build conftests
```

总共有四个文件，都在 `./tests/conformance` 目录下：

- binding_test.go
- pubsub_test.go
- secretstores_test.go
- state_test.go

```makefile
################################################################################
# Target: conf-tests                                                           #
################################################################################
.PHONY: conf-tests
conf-tests:
	CGO_ENABLED=$(CGO) go test -v -tags=conftests -count=1 ./tests/conformance
```

但这些测试在本地是跑不起来的，因为缺乏对应的可连接的外部组件如 azure.eventhubs 、 azure.cosmosdb。

### Target: e2e-tests-zeebe 

端到端测试，类似：

```makefile
################################################################################
# Target: e2e-tests-zeebe                                                      #
################################################################################
.PHONY: e2e-tests-zeebe
e2e-tests-zeebe:
	CGO_ENABLED=$(CGO) go test -v -tags=e2etests -count=1 ./tests/e2e/bindings/zeebe/...
```





