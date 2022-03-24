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
export DAPR_TEST_NAMESPACE=dapr-tests
export TARGET_OS=linux
export TARGET_ARCH=arm64  # 默认是amd64，m1上本地运行需要修改为arm64
# export GOOS=linux
# export GOARCH=amd64
```

#### m1 交叉测试



#### amd64

在 amd64 机器上：

```bash
export DAPR_REGISTRY=docker.io/skyao
export DAPR_TAG=dev
export DAPR_TEST_NAMESPACE=dapr-tests
export TARGET_OS=linux
export TARGET_ARCH=amd64  # 默认是amd64，m1上需要修改
```



### 构建dapr镜像

```bash
make build-linux
make docker-build
make docker-push
```



#### m1 本地测试



#### m1 交叉测试



#### amd64






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

#### m1 本地

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

暂时作罢，这需要改动部署文件了，还不知道 redis 有没有支持 arm64 的镜像。
