---
title: "性能测试部分"
linkTitle: "性能测试"
weight: 200
date: 2022-03-24
description: >
  和性能测试相关的make target
---

性能测试的 target 定义在 dapr_tests.mk：

https://github.com/dapr/dapr/blob/master/tests/dapr_tests.mk

但因为内容比较独立，单独拿出来细看

## 变量定义

PERF_TEST_APPS 定义了性能测试要用到的app：

```properties
# PERFORMANCE test app list
PERF_TEST_APPS=actorfeatures actorjava tester service_invocation_http
```

PERF_TESTAPP_DIR 是性能测试应用所在的目录：

```properties
# PERFORMANCE test app root directory
PERF_TESTAPP_DIR=./tests/apps/perf
```

PERF_TESTS 是性能测试包含的测试案例，这些测试案例在 `tests/perf` 下：

```properties
# PERFORMANCE tests
PERF_TESTS=actor_timer actor_reminder actor_activation service_invocation_http
```



## Target: build-perf-app-x



```makefile
define genPerfTestAppImageBuild
.PHONY: build-perf-app-$(1)
build-perf-app-$(1): check-e2e-env
	$(DOCKER) build -f $(PERF_TESTAPP_DIR)/$(1)/$(DOCKERFILE) $(PERF_TESTAPP_DIR)/$(1)/. -t $(DAPR_TEST_REGISTRY)/perf-$(1):$(DAPR_TEST_TAG)
endef

# Generate perf app image build targets
$(foreach ITEM,$(PERF_TEST_APPS),$(eval $(call genPerfTestAppImageBuild,$(ITEM))))
```



## Target: perf-build-deploy-run



```makefile
perf-build-deploy-run: create-test-namespace setup-3rd-party build docker-push docker-deploy-k8s setup-test-components build-perf-app-all push-perf-app-all test-perf-all
```



## Target: push-perf-app-x



```makefile
define genPerfAppImagePush
.PHONY: push-perf-app-$(1)
push-perf-app-$(1): check-e2e-env
	$(DOCKER) push $(DAPR_TEST_REGISTRY)/perf-$(1):$(DAPR_TEST_TAG)
endef

# Generate perf app image push targets
$(foreach ITEM,$(PERF_TEST_APPS),$(eval $(call genPerfAppImagePush,$(ITEM))))

```



## Target: push-kind-perf-app-x



```makefile
define genPerfAppImageKindPush
.PHONY: push-kind-perf-app-$(1)
push-kind-perf-app-$(1): check-e2e-env
	kind load docker-image $(DAPR_TEST_REGISTRY)/perf-$(1):$(DAPR_TEST_TAG)
endef

# Generate perf app image kind push targets
$(foreach ITEM,$(PERF_TEST_APPS),$(eval $(call genPerfAppImageKindPush,$(ITEM))))
```





## 枚举性能测试target列表

```makefile
# Enumerate test app build targets
BUILD_PERF_APPS_TARGETS:=$(foreach ITEM,$(PERF_TEST_APPS),build-perf-app-$(ITEM))
# Enumerate perf app push targets
PUSH_PERF_APPS_TARGETS:=$(foreach ITEM,$(PERF_TEST_APPS),push-perf-app-$(ITEM))
# Enumerate perf app kind push targets
PUSH_KIND_PERF_APPS_TARGETS:=$(foreach ITEM,$(PERF_TEST_APPS),push-kind-perf-app-$(ITEM))
```





## Target：build-perf-app-all



```makefile
# build perf app image
build-perf-app-all: $(BUILD_PERF_APPS_TARGETS)
```



## Target：push-perf-app-all



```makefile
# push perf app image to the registry
push-perf-app-all: $(PUSH_PERF_APPS_TARGETS)
```



## Target：push-kind-perf-app-all



```makefile
# push perf app image to kind cluster
push-kind-perf-app-all: $(PUSH_KIND_PERF_APPS_TARGETS)
```



## Target: test-perf-x



```makefile
define genPerfTestRun
.PHONY: test-perf-$(1)
test-perf-$(1): check-e2e-env test-deps
	DAPR_CONTAINER_LOG_PATH=$(DAPR_CONTAINER_LOG_PATH) GOOS=$(TARGET_OS_LOCAL) DAPR_TEST_NAMESPACE=$(DAPR_TEST_NAMESPACE) DAPR_TEST_TAG=$(DAPR_TEST_TAG) DAPR_TEST_REGISTRY=$(DAPR_TEST_REGISTRY) DAPR_TEST_MINIKUBE_IP=$(MINIKUBE_NODE_IP) gotestsum --jsonfile $(TEST_OUTPUT_FILE_PREFIX)_perf_$(1).json --junitfile $(TEST_OUTPUT_FILE_PREFIX)_perf_$(1).xml --format standard-quiet -- -timeout 1h -p 1 -count=1 -v -tags=perf ./tests/perf/$(1)/...
	jq -r .Output $(TEST_OUTPUT_FILE_PREFIX)_perf_$(1).json | strings
endef

# Generate perf app image build targets
$(foreach ITEM,$(PERF_TESTS),$(eval $(call genPerfTestRun,$(ITEM))))

TEST_PERF_TARGETS:=$(foreach ITEM,$(PERF_TESTS),test-perf-$(ITEM))
```





## Target: test-perf-all

```makefile
# start all perf tests
test-perf-all: check-e2e-env test-deps
	DAPR_CONTAINER_LOG_PATH=$(DAPR_CONTAINER_LOG_PATH) GOOS=$(TARGET_OS_LOCAL) DAPR_TEST_NAMESPACE=$(DAPR_TEST_NAMESPACE) DAPR_TEST_TAG=$(DAPR_TEST_TAG) DAPR_TEST_REGISTRY=$(DAPR_TEST_REGISTRY) DAPR_TEST_MINIKUBE_IP=$(MINIKUBE_NODE_IP) gotestsum --jsonfile $(TEST_OUTPUT_FILE_PREFIX)_perf.json --junitfile $(TEST_OUTPUT_FILE_PREFIX)_perf.xml --format standard-quiet -- -p 1 -count=1 -v -tags=perf ./tests/perf/...
	jq -r .Output $(TEST_OUTPUT_FILE_PREFIX)_perf.json | strings
```



### m1上执行

执行结果输出如下：

```bash
$ make test-perf-all                  
# The desire here is to download this test dependency without polluting go.mod
# In golang >=1.16 there is a new way to do this with `go install gotest.tools/gotestsum@latest`
# But this doesn't work with <=1.15.
# (see: https://golang.org/ref/mod#go-install)
command -v gotestsum || go install gotest.tools/gotestsum@latest
/Users/sky/work/soft/gopath/bin/gotestsum
DAPR_CONTAINER_LOG_PATH=./dist/container_logs GOOS=darwin DAPR_TEST_NAMESPACE=dapr-system DAPR_TEST_TAG=dev-linux-arm64 DAPR_TEST_REGISTRY=docker.io/skyao DAPR_TEST_MINIKUBE_IP= gotestsum --jsonfile ./test_report_perf.json --junitfile ./test_report_perf.xml --format standard-quiet -- -p 1 -count=1 -v -tags=perf ./tests/perf/...
?       github.com/dapr/dapr/tests/perf [no test files]
2022/03/24 10:52:06 Running setup...
2022/03/24 10:52:06 Installing test apps...
2022/03/24 10:52:06 Adding app {testapp 3000  map[] true perf-actorjava:dev-linux-arm64  docker.io/skyao 1 true true   4.0 0.1 800Mi 2500Mi 4.0 0.1 512Mi 250Mi <nil> false}
2022/03/24 10:52:06 Adding app {tester 3001  map[] true perf-tester:dev-linux-arm64  docker.io/skyao 1 true true   4.0 0.1 800Mi 2500Mi 4.0 0.1 512Mi 250Mi <nil> false}
2022/03/24 10:52:06 Installing apps ...
2022/03/24 10:52:08 Deploying app testapp ...

2022/03/24 11:02:10 deployment testapp relate pods: ......
CST,LastTransitionTime:2022-03-24 11:02:10 +0800 CST,},},ReadyReplicas:0,CollisionCount:nil,},} pod status: map[testapp-7cc8965d47-fzqhs:[]] error: timed out waiting for the condition2022/03/24 11:02:10 Running teardown...
FAIL    github.com/dapr/dapr/tests/perf/actor_activation        605.705s

```



