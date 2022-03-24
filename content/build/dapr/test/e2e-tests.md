---
title: "端到端测试部分"
linkTitle: "端到端测试"
weight: 100
date: 2022-03-24
description: >
  和端到端测试相关的make target
---

E2e测试的 target 定义在 dapr_tests.mk：

https://github.com/dapr/dapr/blob/master/tests/dapr_tests.mk

但因为内容比较独立，单独拿出来细看

## 变量定义

E2E_TEST_APPS 定义了要 e2e 测试要用到的app：

```makefile
# E2E test app list
# e.g. E2E_TEST_APPS=hellodapr state service_invocation
E2E_TEST_APPS=actorjava \
actordotnet \
actorpython \
actorphp \
hellodapr \
stateapp \
secretapp \
service_invocation \
service_invocation_grpc \
service_invocation_grpc_proxy_client \
service_invocation_grpc_proxy_server \
binding_input \
binding_input_grpc \
binding_output \
pubsub-publisher \
pubsub-subscriber \
pubsub-subscriber_grpc \
pubsub-subscriber-routing \
pubsub-subscriber-routing_grpc \
actorapp \
actorclientapp \
actorfeatures \
actorinvocationapp \
actorreentrancy \
runtime \
runtime_init \
middleware \
job-publisher \
```

E2E_TESTAPP_DIR 是 e2e 测试应用所在的目录：

```makefile
# E2E test app root directory
E2E_TESTAPP_DIR=./tests/apps
```





## Target: check-e2e-env

```makefile
# check the required environment variables
check-e2e-env:
ifeq ($(DAPR_TEST_REGISTRY),)
	$(error DAPR_TEST_REGISTRY environment variable must be set)
endif
ifeq ($(DAPR_TEST_TAG),)
	$(error DAPR_TEST_TAG environment variable must be set)
endif
```



## Target: build-e2e-app-x



```makefile
define genTestAppImageBuild
.PHONY: build-e2e-app-$(1)
build-e2e-app-$(1): check-e2e-env
ifeq (,$(wildcard $(E2E_TESTAPP_DIR)/$(1)/$(DOCKERFILE)))
	CGO_ENABLED=0 GOOS=$(TARGET_OS) GOARCH=$(TARGET_ARCH) go build -o $(E2E_TESTAPP_DIR)/$(1)/app$(BINARY_EXT_LOCAL) $(E2E_TESTAPP_DIR)/$(1)/app.go
	$(DOCKER) build -f $(E2E_TESTAPP_DIR)/$(DOCKERFILE) $(E2E_TESTAPP_DIR)/$(1)/. -t $(DAPR_TEST_REGISTRY)/e2e-$(1):$(DAPR_TEST_TAG)
else
	$(DOCKER) build -f $(E2E_TESTAPP_DIR)/$(1)/$(DOCKERFILE) $(E2E_TESTAPP_DIR)/$(1)/. -t $(DAPR_TEST_REGISTRY)/e2e-$(1):$(DAPR_TEST_TAG)
endif
endef

# Generate test app image build targets
$(foreach ITEM,$(E2E_TEST_APPS),$(eval $(call genTestAppImageBuild,$(ITEM))))
```



## Target: push-e2e-app-x



```makefile
define genTestAppImagePush
.PHONY: push-e2e-app-$(1)
push-e2e-app-$(1): check-e2e-env
	$(DOCKER) push $(DAPR_TEST_REGISTRY)/e2e-$(1):$(DAPR_TEST_TAG)
endef

# Generate test app image push targets
$(foreach ITEM,$(E2E_TEST_APPS),$(eval $(call genTestAppImagePush,$(ITEM))))
```



## Target: push-kind-e2e-app



```makefile
define genTestAppImageKindPush
.PHONY: push-kind-e2e-app-$(1)
push-kind-e2e-app-$(1): check-e2e-env
	kind load docker-image $(DAPR_TEST_REGISTRY)/e2e-$(1):$(DAPR_TEST_TAG)
endef

# Generate test app image push targets
$(foreach ITEM,$(E2E_TEST_APPS),$(eval $(call genTestAppImageKindPush,$(ITEM))))
```



## Target: e2e-build-deploy-run



```makefile
e2e-build-deploy-run: create-test-namespace setup-3rd-party build docker-push docker-deploy-k8s setup-test-components build-e2e-app-all push-e2e-app-all test-e2e-all
```



## 枚举 e2e 测试target列表

```makefile
# Enumerate test app build targets
BUILD_E2E_APPS_TARGETS:=$(foreach ITEM,$(E2E_TEST_APPS),build-e2e-app-$(ITEM))
# Enumerate test app push targets
PUSH_E2E_APPS_TARGETS:=$(foreach ITEM,$(E2E_TEST_APPS),push-e2e-app-$(ITEM))
# Enumerate test app push targets
PUSH_KIND_E2E_APPS_TARGETS:=$(foreach ITEM,$(E2E_TEST_APPS),push-kind-e2e-app-$(ITEM))
```



## Target：build-e2e-app-all



```makefile
# build test app image
build-e2e-app-all: $(BUILD_E2E_APPS_TARGETS)
```



## Target：push-e2e-app-all



```makefile
# push test app image to the registry
push-e2e-app-all: $(PUSH_E2E_APPS_TARGETS)
```



## Target：push-kind-e2e-app-all



```makefile
# push test app image to kind cluster
push-kind-e2e-app-all: $(PUSH_KIND_E2E_APPS_TARGETS)
```



## Target: test-e2e-all



```makefile
# start all e2e tests
test-e2e-all: check-e2e-env test-deps
	# Note: we can set -p 2 to run two tests apps at a time, because today we do not share state between
	# tests. In the future, if we add any tests that modify global state (such as dapr config), we'll 
	# have to be sure and run them after the main test suite, so as not to alter the state of a running
	# test
	# Note2: use env variable DAPR_E2E_TEST to pick one e2e test to run.
     ifeq ($(DAPR_E2E_TEST),)
	DAPR_CONTAINER_LOG_PATH=$(DAPR_CONTAINER_LOG_PATH) GOOS=$(TARGET_OS_LOCAL) DAPR_TEST_NAMESPACE=$(DAPR_TEST_NAMESPACE) DAPR_TEST_TAG=$(DAPR_TEST_TAG) DAPR_TEST_REGISTRY=$(DAPR_TEST_REGISTRY) DAPR_TEST_MINIKUBE_IP=$(MINIKUBE_NODE_IP) gotestsum --jsonfile $(TEST_OUTPUT_FILE_PREFIX)_e2e.json --junitfile $(TEST_OUTPUT_FILE_PREFIX)_e2e.xml --format standard-quiet -- -p 2 -count=1 -v -tags=e2e ./tests/e2e/$(DAPR_E2E_TEST)/...
     else
	for app in $(DAPR_E2E_TEST); do \
		DAPR_CONTAINER_LOG_PATH=$(DAPR_CONTAINER_LOG_PATH) GOOS=$(TARGET_OS_LOCAL) DAPR_TEST_NAMESPACE=$(DAPR_TEST_NAMESPACE) DAPR_TEST_TAG=$(DAPR_TEST_TAG) DAPR_TEST_REGISTRY=$(DAPR_TEST_REGISTRY) DAPR_TEST_MINIKUBE_IP=$(MINIKUBE_NODE_IP) gotestsum --jsonfile $(TEST_OUTPUT_FILE_PREFIX)_e2e.json --junitfile $(TEST_OUTPUT_FILE_PREFIX)_e2e.xml --format standard-quiet -- -p 2 -count=1 -v -tags=e2e ./tests/e2e/$$app/...; \
	done
     endif
```





