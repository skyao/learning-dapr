---
type: docs
title: "e2e测试的搭建"
linkTitle: "搭建"
weight: 4100
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
- 准备golang：必须是go 1.16
- 准备k8s

### 准备helm v3

参考：https://helm.sh/docs/intro/install/

在mac下最简单的方式就是用 brew 安装：

```bash
brew install helm

helm version
version.BuildInfo{Version:"v3.5.4", GitCommit:"1b5edb69df3d3a08df77c9902dc17af864ff05d1", GitTreeState:"dirty", GoVersion:"go1.16.3"}
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

```bash
export DAPR_REGISTRY=docker.io/your_dockerhub_id
export DAPR_TAG=dev
export DAPR_NAMESPACE=dapr-tests
```

### 构建dapr镜像

```bash
make build-linux
make docker-build
make docker-push
```




## 部署e2e测试的应用

参考文档指示，一步一步来：

https://github.com/dapr/dapr/blob/master/tests/docs/running-e2e-test.md#option-2-step-by-step-guide

### 准备测试用的namespace

准备测试用的 namespace，在 `dapr/dapr` 仓库下执行：

```bash
make delete-test-namespace
make create-test-namespace

kubectl create namespace dapr-tests
namespace/dapr-tests created
kubectl create namespace dapr-tests-2
namespace/dapr-tests-2 created
```

实际上会构建两个 namespace，这个 dapr-tests-2 应该是后面加的，暂时不还不知道用在什么情况下。

### 初始化heml

```bash
make setup-helm-init
```

### 准备测试会用到的redis和kafka

```bash
make setup-test-env-redis
make setup-test-env-kafka
```


Containers:
  dapr-sidecar-injector:
    Container ID:  docker://596245f5fe624a24ee169a6cf92d908a8ca8800e93b89290078b9821faa0e250
    Image:         docker.io/skyao/dapr:dev-linux-amd64
    Image ID:      docker-pullable://skyao/dapr@sha256:c3cb8d57b611207efda277dfced83ec8417b864cc3c4dc9f2398d6cefe877dbe
    
    
Containers:
  dapr-sidecar-injector:
    Container ID:  docker://9ec86d2637893c140f6775039d40553e0763a3978b46d2ed9e61ddc5c1ee5308
    Image:         docker.io/skyao/dapr:dev-linux-amd64
    Image ID:      docker-pullable://skyao/dapr@sha256:63aceee9d73de93b398f1e87c0dd2aaf862dd7b3ea0a73aa96e6de8b11819c07
    Ports:         4000/TCP, 9090/TCP
    Host Ports:    0/TCP, 0/TCP

