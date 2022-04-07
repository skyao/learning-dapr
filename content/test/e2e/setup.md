---
title: "e2e测试的搭建"
linkTitle: "搭建"
weight: 100
date: 2021-05-09
description: >
  搭建Dapr的e2e测试
---

## 前言

dapr 为 e2e 测试提供了一个说明文档， 存在放 `dapr/dapr` 仓库下的 `tests/docs/running-e2e-test.md ` 下：

https://github.com/dapr/dapr/blob/master/tests/docs/running-e2e-test.md

以下内容为参考官方文档的详细步骤。

## 准备工作

### 准备dapr开发环境

https://github.com/dapr/dapr/blob/master/docs/development/setup-dapr-development-env.md

具体有：

- 准备docker：包括安装 docker ，创建 docker hub 账号
- 准备golang：必须是go 1.17
- 准备k8s

### 准备helm v3

参考：https://helm.sh/docs/intro/install/

在mac下最简单的方式就是用 brew 安装：

```bash
$ brew install helm

$ helm version
version.BuildInfo{Version:"v3.8.1", GitCommit:"5cb9af4b1b271d11d7a97a71df3ac337dd94ad37", GitTreeState:"clean", GoVersion:"go1.17.8"}
```

### 准备dapr

e2e 需要用到 kubernetes。

- 安装dapr命令行

  参考： [Install the Dapr CLI | Dapr Docs](https://docs.dapr.io/getting-started/install-dapr-cli/)

- ~~在 kubernets 中安装 dapr 控制面~~

  参考：[quickstarts/hello-kubernetes at master · dapr/quickstarts (github.com)](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes)

  ```bash
  dapr init --kubernetes --wait
  ```

  这样安装出来的是标准的dapr，其镜像版本为release版本，如下所示， dapr-sidecar-injector pod的container：

  ```bash
  Containers:
    dapr-sidecar-injector:
      Container ID:  docker://bbda12f7c31adc380122651ca2b0ccf90cc599735da54df411df5764689e4cd6
      Image:         docker.io/daprio/dapr:1.1.2
  ```

  所以，如果需要改动dapr控制面，则不能用这个方式安装dapr控制面。如果只是测试验证daprd（sidecar），则这个方式是可以的。

## 搭建 Dapr 控制面

以master分支为例，从dapr/dapr仓库的源码开始构建。

### 设置e2e相关的环境变量

#### m1本地测试

在 m1 macbook 上，如果为了构建后给本地的 arm64 docker 和 arm64 k8s 运行：

```bash
export DAPR_REGISTRY=docker.io/skyao
export DAPR_TAG=dev
export DAPR_NAMESPACE=dapr-tests
export DAPR_TEST_NAMESPACE=dapr-tests
export DAPR_TEST_REGISTRY=docker.io/skyao
export DAPR_TEST_TAG=dev-linux-amd64
export TARGET_OS=linux
export TARGET_ARCH=arm64  # 默认是amd64，m1上本地运行需要修改为arm64
# export GOOS=linux
# export GOARCH=amd64
```

#### m1 交叉测试

在 m1 macbook 上，如果为了构建后给远程的 amd64 docker 和 amd64 k8s 运行：

```bash
export DAPR_REGISTRY=docker.io/skyao
export DAPR_TAG=dev
export DAPR_NAMESPACE=dapr-tests
export DAPR_TEST_NAMESPACE=dapr-tests
export DAPR_TEST_REGISTRY=docker.io/skyao
export DAPR_TEST_TAG=dev-linux-amd64
export GOOS=linux
export GOARCH=amd64
```

#### amd64

在 amd64 机器上：

```bash
export DAPR_REGISTRY=docker.io/skyao
export DAPR_TAG=dev
export DAPR_NAMESPACE=dapr-tests
export DAPR_TEST_NAMESPACE=dapr-tests
export DAPR_TEST_REGISTRY=docker.io/skyao
export DAPR_TEST_TAG=dev-linux-amd64
export GOOS=linux
export GOARCH=amd64
```

### 构建dapr镜像

```bash
make build-linux
make docker-build
make docker-push
```



#### ~~m1 本地测试~~

由于缺乏类似 redis 之类的 arm64 镜像, 这个方案跑不通.

暂时放弃.

#### m1 交叉测试

在 m1 macbook 上构建 linux + amd 64 的二进制文件：

```bash
$ make build-linux 

CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f-dirty -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_amd64/release/daprd ./cmd/daprd/;
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f-dirty -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_amd64/release/placement ./cmd/placement/;
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f-dirty -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_amd64/release/operator ./cmd/operator/;
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f-dirty -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_amd64/release/injector ./cmd/injector/;
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f-dirty -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_amd64/release/sentry ./cmd/sentry/;
```

打包 docker 镜像：

```bash
$ make docker-build     

Building docker.io/skyao/dapr:dev docker image ...
docker build --build-arg PKG_FILES=* -f ./docker/Dockerfile ./dist/linux_amd64/release -t docker.io/skyao/dapr:dev-linux-amd64
[+] Building 16.1s (12/12) FINISHED    
......
 => => naming to docker.io/skyao/dapr:dev-linux-amd64 
 => => naming to docker.io/skyao/daprd:dev-linux-amd64 
 => => naming to docker.io/skyao/placement:dev-linux-amd64 
 => => naming to docker.io/skyao/sentry:dev-linux-amd64 
```

推送 docker 镜像

```bash
$ make docker-push

Building docker.io/skyao/dapr:dev docker image ...
docker build --build-arg PKG_FILES=* -f ./docker/Dockerfile ./dist/linux_amd64/release -t docker.io/skyao/dapr:dev-linux-amd64
.....

docker push docker.io/skyao/dapr:dev-linux-amd64
The push refers to repository [docker.io/skyao/dapr]

docker push docker.io/skyao/daprd:dev-linux-amd64
The push refers to repository [docker.io/skyao/daprd]

docker push docker.io/skyao/placement:dev-linux-amd64
The push refers to repository [docker.io/skyao/placement]

docker push docker.io/skyao/sentry:dev-linux-amd64
The push refers to repository [docker.io/skyao/sentry]


```



#### amd64

在 x86 机器上构建 linux + amd 64 的二进制文件：

```bash
$ make build-linux 

CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=79ffea446dfd14ac25429c8dc9ad792983b46ce2 -X github.com/dapr/dapr/pkg/version.gitversion=v1.0.0-rc.4-1305-g79ffea4 -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_amd64/release/daprd ./cmd/daprd/;
go: downloading github.com/dapr/components-contrib v1.6.0-rc.2.0.20220322152414-4d44c2f04f43
go: downloading github.com/nats-io/nats.go v1.13.1-0.20220308171302-2f2f6968e98d
go: downloading github.com/alibabacloud-go/darabonba-openapi v0.1.16
go: downloading github.com/apache/pulsar-client-go v0.8.1
go: downloading github.com/prometheus/client_golang v1.11.1
go: downloading github.com/klauspost/compress v1.14.4
go: downloading github.com/alibabacloud-go/alibabacloud-gateway-spi v0.0.4
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=79ffea446dfd14ac25429c8dc9ad792983b46ce2 -X github.com/dapr/dapr/pkg/version.gitversion=v1.0.0-rc.4-1305-g79ffea4 -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_amd64/release/placement ./cmd/placement/;
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=79ffea446dfd14ac25429c8dc9ad792983b46ce2 -X github.com/dapr/dapr/pkg/version.gitversion=v1.0.0-rc.4-1305-g79ffea4 -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_amd64/release/operator ./cmd/operator/;
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=79ffea446dfd14ac25429c8dc9ad792983b46ce2 -X github.com/dapr/dapr/pkg/version.gitversion=v1.0.0-rc.4-1305-g79ffea4 -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_amd64/release/injector ./cmd/injector/;
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=79ffea446dfd14ac25429c8dc9ad792983b46ce2 -X github.com/dapr/dapr/pkg/version.gitversion=v1.0.0-rc.4-1305-g79ffea4 -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_amd64/release/sentry ./cmd/sentry/;
```

打包 docker 镜像：

```bash
$ make docker-build     

Building docker.io/skyao/dapr:dev docker image ...
docker build --build-arg PKG_FILES=* -f ./docker/Dockerfile ./dist/linux_amd64/release -t docker.io/skyao/dapr:dev-linux-amd64
Sending build context to Docker daemon  236.2MB

......
Successfully tagged skyao/dapr:dev-linux-amd64
Successfully tagged skyao/daprd:dev-linux-amd64
Successfully tagged skyao/placement:dev-linux-amd64
Successfully tagged skyao/sentry:dev-linux-amd64
```

推送 docker 镜像

```bash
$ make docker-push

Building docker.io/skyao/dapr:dev docker image ...
docker build --build-arg PKG_FILES=* -f ./docker/Dockerfile ./dist/linux_amd64/release -t docker.io/skyao/dapr:dev-linux-amd64
.....

docker push docker.io/skyao/dapr:dev-linux-amd64
The push refers to repository [docker.io/skyao/dapr]

docker push docker.io/skyao/daprd:dev-linux-amd64
The push refers to repository [docker.io/skyao/daprd]

docker push docker.io/skyao/placement:dev-linux-amd64
The push refers to repository [docker.io/skyao/placement]

docker push docker.io/skyao/sentry:dev-linux-amd64
The push refers to repository [docker.io/skyao/sentry]


```

如果执行时遇到报错:

```bash
denied: requested access to the resource is denied
make: *** [docker/docker.mk:110: docker-push] Error 1
```

可以通过 `docker login` 命令先登录再 push。


## 部署e2e测试的应用

参考文档指示，一步一步来：

https://github.com/dapr/dapr/blob/master/tests/docs/running-e2e-test.md#option-2-step-by-step-guide

### 准备测试用的namespace

准备测试用的 namespace，在 `dapr/dapr` 仓库下执行：

```bash
$ make delete-test-namespace
$ make create-test-namespace

kubectl create namespace dapr-tests
namespace/dapr-tests created
kubectl create namespace dapr-tests-2
namespace/dapr-tests-2 created

# 如果测试的 namespace 不是 dapr-system，则需要手工建立 dapr-system
# 文档里面忽略了这个问题
$ kubectl create namespace dapr-system
```

实际上会构建两个 namespace。

### 初始化heml

```bash
$ make setup-helm-init

helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
helm repo add stable https://charts.helm.sh/stable
"stable" has been added to your repositories
helm repo add incubator https://charts.helm.sh/incubator
"incubator" has been added to your repositories
helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "incubator" chart repository
...Successfully got an update from the "bitnami" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
```

### 准备测试会用到的redis和kafka

```bash
make setup-test-env-redis
make setup-test-env-kafka
```

### 部署 dapr 控制面

```bash
$ make docker-deploy-k8s

Deploying docker.io/skyao/dapr:dev to the current K8S context...
helm install \
                dapr --namespace=dapr-system --wait --timeout 5m0s \
                --set global.ha.enabled=false --set-string global.tag=dev-linux-amd64 \
                --set-string global.registry=docker.io/skyao --set global.logAsJson=true \
                --set global.daprControlPlaneOs=linux --set global.daprControlPlaneArch=amd64 \
                --set dapr_placement.logLevel=debug --set dapr_sidecar_injector.sidecarImagePullPolicy=Always \
                --set global.imagePullPolicy=Always --set global.imagePullSecrets= \
                --set global.mtls.enabled=true \
                --set dapr_placement.cluster.forceInMemoryLog=true ./charts/dapr
NAME: dapr
LAST DEPLOYED: Thu Mar 24 18:43:41 2022
NAMESPACE: dapr-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing Dapr: High-performance, lightweight serverless runtime for cloud and edge

Your release is named dapr.

To get started with Dapr, we recommend using our quickstarts:
https://github.com/dapr/quickstarts

For more information on running Dapr, visit:
https://dapr.io
```

部署之后，检查一下pod 的情况

```bash
$ k get pods -A                 
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
dapr-system   dapr-dashboard-7c99fc9c45-cgnb6          1/1     Running   0          101s
dapr-system   dapr-operator-556dfb5987-xfnwt           1/1     Running   0          101s
dapr-system   dapr-placement-server-0                  1/1     Running   0          101s
dapr-system   dapr-sentry-54cb9989d5-hq5m4             1/1     Running   0          101s
dapr-system   dapr-sidecar-injector-58d99b46c7-vxn5r   1/1     Running   0          101s
dapr-tests    dapr-kafka-0                             1/1     Running   0          18m
dapr-tests    dapr-kafka-zookeeper-0                   1/1     Running   0          18m
dapr-tests    dapr-redis-master-0                      1/1     Running   0          18m
......
```

关闭 mtls:

```bash
make setup-disable-mtls
kubectl apply -f ./tests/config/dapr_mtls_off_config.yaml --namespace dapr-tests
configuration.dapr.io/daprsystem created
```

###  部署测试用的组件

```bash
$ make setup-test-components

kubectl apply -f ./tests/config/dapr_observability_test_config.yaml --namespace dapr-tests
configuration.dapr.io/disable-telemetry created
configuration.dapr.io/obs-defaultmetric created
kubectl apply -f ./tests/config/kubernetes_secret.yaml --namespace dapr-tests
secret/daprsecret created
secret/daprsecret2 created
secret/emptysecret created
kubectl apply -f ./tests/config/kubernetes_secret_config.yaml --namespace dapr-tests
configuration.dapr.io/secretappconfig created
kubectl apply -f ./tests/config/kubernetes_redis_secret.yaml --namespace dapr-tests
secret/redissecret created
kubectl apply -f ./tests/config/dapr_redis_state.yaml --namespace dapr-tests
component.dapr.io/statestore created
kubectl apply -f ./tests/config/dapr_mongodb_state.yaml --namespace dapr-tests
component.dapr.io/querystatestore created
kubectl apply -f ./tests/config/dapr_tests_cluster_role_binding.yaml --namespace dapr-tests
rolebinding.rbac.authorization.k8s.io/dapr-secret-reader created
role.rbac.authorization.k8s.io/secret-reader created
kubectl apply -f ./tests/config/dapr_redis_pubsub.yaml --namespace dapr-tests
component.dapr.io/messagebus created
kubectl apply -f ./tests/config/dapr_kafka_bindings.yaml --namespace dapr-tests
component.dapr.io/test-topic created
kubectl apply -f ./tests/config/dapr_kafka_bindings_custom_route.yaml --namespace dapr-tests
component.dapr.io/test-topic-custom-route created
kubectl apply -f ./tests/config/dapr_kafka_bindings_grpc.yaml --namespace dapr-tests
component.dapr.io/test-topic-grpc created
kubectl apply -f ./tests/config/app_topic_subscription_pubsub.yaml --namespace dapr-tests
subscription.dapr.io/c-topic-subscription created
kubectl apply -f ./tests/config/app_topic_subscription_pubsub_grpc.yaml --namespace dapr-tests
subscription.dapr.io/c-topic-subscription-grpc created
kubectl apply -f ./tests/config/kubernetes_allowlists_config.yaml --namespace dapr-tests
configuration.dapr.io/allowlistsappconfig created
kubectl apply -f ./tests/config/kubernetes_allowlists_grpc_config.yaml --namespace dapr-tests
configuration.dapr.io/allowlistsgrpcappconfig created
kubectl apply -f ./tests/config/dapr_redis_state_query.yaml --namespace dapr-tests
component.dapr.io/querystatestore2 created
kubectl apply -f ./tests/config/dapr_redis_state_badhost.yaml --namespace dapr-tests
component.dapr.io/badhost-store created
kubectl apply -f ./tests/config/dapr_redis_state_badpass.yaml --namespace dapr-tests
component.dapr.io/badpass-store created
kubectl apply -f ./tests/config/uppercase.yaml --namespace dapr-tests
component.dapr.io/uppercase created
kubectl apply -f ./tests/config/pipeline.yaml --namespace dapr-tests
configuration.dapr.io/pipeline created
kubectl apply -f ./tests/config/app_reentrant_actor.yaml --namespace dapr-tests
configuration.dapr.io/reentrantconfig created
kubectl apply -f ./tests/config/app_actor_type_metadata.yaml --namespace dapr-tests
configuration.dapr.io/actortypemetadata created
kubectl apply -f ./tests/config/app_topic_subscription_routing.yaml --namespace dapr-tests
subscription.dapr.io/pubsub-routing-crd-http-subscription created
kubectl apply -f ./tests/config/app_topic_subscription_routing_grpc.yaml --namespace dapr-tests
subscription.dapr.io/pubsub-routing-crd-grpc-subscription created
kubectl apply -f ./tests/config/app_pubsub_routing.yaml --namespace dapr-tests
configuration.dapr.io/pubsubroutingconfig created
# Show the installed components
kubectl get components --namespace dapr-tests
NAME                      AGE
badhost-store             23s
badpass-store             20s
messagebus                49s
querystatestore           56s
querystatestore2          27s
statestore                58s
test-topic                45s
test-topic-custom-route   42s
test-topic-grpc           39s
uppercase                 18s
# Show the installed configurations
kubectl get configurations --namespace dapr-tests
NAME                      AGE
actortypemetadata         11s
allowlistsappconfig       33s
allowlistsgrpcappconfig   31s
daprsystem                93s
disable-telemetry         73s
obs-defaultmetric         72s
pipeline                  16s
pubsubroutingconfig       3s
reentrantconfig           13s
secretappconfig           65s
```



## 部署测试用的应用



```bash
make build-e2e-app-all

make push-e2e-app-all
```

## 运行e2e测试

```bash
$ make test-e2e-all

# Note2: use env variable DAPR_E2E_TEST to pick one e2e test to run.

```



根据提示，如果只想单独执行某个测试案例，可以设置环境变量 DAPR_E2E_TEST：

```bash
$ export DAPR_E2E_TEST=service_invocation

$ make test-e2e-all

```

测试非常不稳定，待查。



## 特殊情况

#### m1 本地启动k8s

部署之后 redis pod 启动不起来，报错 '1 node(s) didn't match Pod's node affinity/selector.'，经检查是因为 redis pods 的设置中要求部署在 linux + amd64 下，m1 本地启动的 k8s 节点是 arm64，所以没有节点可以启动：

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
          - key: kubernetes.io/arch
            operator: In
            values:
            - amd64
```

暂时作罢，这需要改动部署文件了。redis 有 arm64 的镜像：

https://hub.docker.com/r/arm64v8/redis/

试了一下在 m1 上执行 `docker run --name some-redis -d arm64v8/redis` 命令是可以正常在docker中启动 redis 的。

原则上修改pod一下内容应该就可以跑起来：

 ```yaml
           - key: kubernetes.io/arch
             operator: In
             values:
             - amd64  # 修改为 arm64
 
 image: docker.io/redislabs/rejson:latest  # 修改为 image: docker.io/arm64v8/redis:latest
 ```

查了一下 helm 对 redis arm64 还没有提供支持，而且这个需求一年内被提出了n次，但还是唧唧歪歪的不支持。

所以，结论是：没法在 m1 macbook 上启动 k8s 并完成开发测试。

