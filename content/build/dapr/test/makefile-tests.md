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

## 变量定义

容器和k8s相关：

```makefile
KUBECTL=kubectl

DAPR_CONTAINER_LOG_PATH?=./dist/container_logs

DAPR_TEST_SECONDARY_NAMESPACE=dapr-tests-2

# 测试用的 namespace，默认是 dapr-system，这个不合适，一般还是设置为 dapr-test 吧
ifeq ($(DAPR_TEST_NAMESPACE),)
DAPR_TEST_NAMESPACE=$(DAPR_NAMESPACE)
endif

ifeq ($(DAPR_TEST_REGISTRY),)
DAPR_TEST_REGISTRY=$(DAPR_REGISTRY)
endif

# 测试使用的镜像文件的 tag
ifeq ($(DAPR_TEST_TAG),)
DAPR_TEST_TAG=$(DAPR_TAG)-$(TARGET_OS)-$(TARGET_ARCH)
endif

# mimikube相关
ifeq ($(DAPR_TEST_ENV),minikube)
MINIKUBE_NODE_IP=$(shell minikube ip)
ifeq ($(MINIKUBE_NODE_IP),)
$(error cannot find get minikube node ip address. ensure that you have minikube environment.)
endif
endif
```

测试使用的 statue store 和 pub/sub 的默认值：

```makefile
ifeq ($(DAPR_TEST_STATE_STORE),)
DAPR_TEST_STATE_STORE=redis
endif

ifeq ($(DAPR_TEST_QUERY_STATE_STORE),)
DAPR_TEST_QUERY_STATE_STORE=mongodb
endif

ifeq ($(DAPR_TEST_PUBSUB),)
DAPR_TEST_PUBSUB=redis
endif
```

检测当前os：

```makefile
ifeq ($(OS),Windows_NT) 
    detected_OS := windows
else
    detected_OS := $(shell sh -c 'uname 2>/dev/null || echo Unknown' |  tr '[:upper:]' '[:lower:]')
endif
```





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

## Target: create-test-namespace

```makefile
create-test-namespace:
	kubectl create namespace $(DAPR_TEST_NAMESPACE)
	kubectl create namespace $(DAPR_TEST_SECONDARY_NAMESPACE)
```

## Target: delete-test-namespace

```makefile
delete-test-namespace:
	kubectl delete namespace $(DAPR_TEST_NAMESPACE)
	kubectl delete namespace $(DAPR_TEST_SECONDARY_NAMESPACE)
```

## Target: setup-3rd-party

```makefile
setup-3rd-party: setup-helm-init setup-test-env-redis setup-test-env-kafka setup-test-env-mongodb
```



## Target: test-deps



```makefile
.PHONY: test-deps
test-deps:
	# The desire here is to download this test dependency without polluting go.mod
	# In golang >=1.16 there is a new way to do this with `go install gotest.tools/gotestsum@latest`
	# But this doesn't work with <=1.15.
	# (see: https://golang.org/ref/mod#go-install)
	command -v gotestsum || go install gotest.tools/gotestsum@latest

```



## Target: setup-helm-init



```makefile
# add required helm repo
setup-helm-init:
	$(HELM) repo add bitnami https://charts.bitnami.com/bitnami
	$(HELM) repo add stable https://charts.helm.sh/stable
	$(HELM) repo add incubator https://charts.helm.sh/incubator
	$(HELM) repo update

```



## redis 相关的target



```makefile
# install redis to the cluster without password
setup-test-env-redis:
	$(HELM) install dapr-redis bitnami/redis --wait --timeout 5m0s --namespace $(DAPR_TEST_NAMESPACE) -f ./tests/config/redis_override.yaml

delete-test-env-redis:
	${HELM} del dapr-redis --namespace ${DAPR_TEST_NAMESPACE}
```

## kafka 相关的target



```makefile
# install kafka to the cluster
setup-test-env-kafka:
	$(HELM) install dapr-kafka bitnami/kafka -f ./tests/config/kafka_override.yaml --namespace $(DAPR_TEST_NAMESPACE) --timeout 10m0s

# delete kafka from cluster
delete-test-env-kafka:
	$(HELM) del dapr-kafka --namespace $(DAPR_TEST_NAMESPACE) 
```


## mongodb 相关的target



```makefile
# install mongodb to the cluster without password
setup-test-env-mongodb:
	$(HELM) install dapr-mongodb bitnami/mongodb -f ./tests/config/mongodb_override.yaml --namespace $(DAPR_TEST_NAMESPACE) --wait --timeout 5m0s

# delete mongodb from cluster
delete-test-env-mongodb:
	${HELM} del dapr-mongodb --namespace ${DAPR_TEST_NAMESPACE}
```


## Target: setup-test-env

```makefile
# Install redis and kafka to test cluster
setup-test-env: setup-test-env-kafka setup-test-env-redis setup-test-env-mongodb
```

## 保存k8s 资源和日志

```makefile
save-dapr-control-plane-k8s-resources:
	mkdir -p '$(DAPR_CONTAINER_LOG_PATH)'
	kubectl describe all -n $(DAPR_TEST_NAMESPACE) > '$(DAPR_CONTAINER_LOG_PATH)/control_plane_k8s_resources.txt'

save-dapr-control-plane-k8s-logs:
	mkdir -p '$(DAPR_CONTAINER_LOG_PATH)'
	kubectl logs -l 'app.kubernetes.io/name=dapr' -n $(DAPR_TEST_NAMESPACE) > '$(DAPR_CONTAINER_LOG_PATH)/control_plane_containers.log'

```



## Target: setup-disable-mtls



```makefile
# Apply default config yaml to turn mTLS off for testing (mTLS is enabled by default)
setup-disable-mtls:
	$(KUBECTL) apply -f ./tests/config/dapr_mtls_off_config.yaml --namespace $(DAPR_TEST_NAMESPACE)
```



## Target: setup-app-configurations



```makefile
# Apply default config yaml to turn tracing off for testing (tracing is enabled by default)
setup-app-configurations:
	$(KUBECTL) apply -f ./tests/config/dapr_observability_test_config.yaml --namespace $(DAPR_TEST_NAMESPACE)
```



## Target: setup-test-components



```makefile
# Apply component yaml for state, secrets, pubsub, and bindings
setup-test-components: setup-app-configurations
	$(KUBECTL) apply -f ./tests/config/kubernetes_secret.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/kubernetes_secret_config.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/kubernetes_redis_secret.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/dapr_$(DAPR_TEST_STATE_STORE)_state.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/dapr_$(DAPR_TEST_QUERY_STATE_STORE)_state.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/dapr_tests_cluster_role_binding.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/dapr_$(DAPR_TEST_PUBSUB)_pubsub.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/dapr_kafka_bindings.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/dapr_kafka_bindings_custom_route.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/dapr_kafka_bindings_grpc.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/app_topic_subscription_pubsub.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/app_topic_subscription_pubsub_grpc.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/kubernetes_allowlists_config.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/kubernetes_allowlists_grpc_config.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/dapr_redis_state_query.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/dapr_redis_state_badhost.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/dapr_redis_state_badpass.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/uppercase.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/pipeline.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/app_reentrant_actor.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/app_actor_type_metadata.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/app_topic_subscription_routing.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/app_topic_subscription_routing_grpc.yaml --namespace $(DAPR_TEST_NAMESPACE)
	$(KUBECTL) apply -f ./tests/config/app_pubsub_routing.yaml --namespace $(DAPR_TEST_NAMESPACE)

	# Show the installed components
	$(KUBECTL) get components --namespace $(DAPR_TEST_NAMESPACE)

	# Show the installed configurations
	$(KUBECTL) get configurations --namespace $(DAPR_TEST_NAMESPACE)

```



## Target: clean-test-env



```makefile
# Clean up test environment
clean-test-env:
	./tests/test-infra/clean_up.sh $(DAPR_TEST_NAMESPACE)
	./tests/test-infra/clean_up.sh $(DAPR_TEST_NAMESPACE)-2
```

TODO: 这里应该用 DAPR_TEST_SECONDARY_NAMESPACE ， 随手 PR： [fix build target clean-test-env by skyao · Pull Request #4420 · dapr/dapr (github.com)](https://github.com/dapr/dapr/pull/4420)



## kind 相关的 target



```makefile

# Setup kind
setup-kind:
	kind create cluster --config ./tests/config/kind.yaml --name kind
	kubectl cluster-info --context kind-kind
	# Setup registry
	docker run -d --restart=always -p 5000:5000 --name kind-registry registry:2
	# Connect the registry to the KinD network.
	docker network connect "kind" kind-registry
	# Setup metrics-server
	helm install ms stable/metrics-server -n kube-system --set=args={--kubelet-insecure-tls}

describe-kind-env:
	@echo "\
	export MINIKUBE_NODE_IP=`kubectl get nodes \
	    -lkubernetes.io/hostname!=kind-control-plane \
        -ojsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}'`\n\
	export DAPR_REGISTRY=$${DAPR_REGISTRY:-localhost:5000/dapr}\n\
	export DAPR_TEST_REGISTRY=$${DAPR_TEST_REGISTRY:-localhost:5000/dapr}\n\
	export DAPR_TAG=dev\n\
	export DAPR_NAMESPACE=dapr-tests"
	

delete-kind:
	docker stop kind-registry && docker rm kind-registry || echo "Could not delete registry."
	kind delete cluster --name kind

```



## minikube 相关的 target



```makefile
setup-minikube-darwin:
	minikube start --memory=4g --cpus=4 --driver=hyperkit --kubernetes-version=v1.18.8
	minikube addons enable metrics-server

setup-minikube-windows:
	minikube start --memory=4g --cpus=4 --kubernetes-version=v1.18.8
	minikube addons enable metrics-server

setup-minikube-linux:
	minikube start --memory=4g --cpus=4 --kubernetes-version=v1.18.8
	minikube addons enable metrics-server

setup-minikube: setup-minikube-$(detected_OS)

describe-minikube-env:
	@echo "\
	export DAPR_REGISTRY=docker.io/`docker-credential-desktop list | jq -r '\
	. | to_entries[] | select(.key | contains("docker.io")) | last(.value)'`\n\
	export DAPR_TAG=dev\n\
	export DAPR_NAMESPACE=dapr-tests\n\
	export DAPR_TEST_ENV=minikube\n\
	export DAPR_TEST_REGISTRY=\n\
	export MINIKUBE_NODE_IP="

# Setup minikube
delete-minikube:
	minikube delete
```

