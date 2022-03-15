---
title: "Makefile docker部分"
linkTitle: "docker"
weight: 40
date: 2022-03-11
description: >
  和 docker 相关的 make target
---

https://github.com/dapr/dapr/blob/master/Makefile

```makefile
################################################################################
# Target: docker                                                               #
################################################################################
include docker/docker.mk
```

https://github.com/dapr/dapr/blob/master/docker/docker.mk

## 变量定义

image相关的变量定义：

```makefile
# Docker image build and push setting
DOCKER:=docker
DOCKERFILE_DIR?=./docker

DAPR_SYSTEM_IMAGE_NAME=$(RELEASE_NAME)
DAPR_RUNTIME_IMAGE_NAME=daprd
DAPR_PLACEMENT_IMAGE_NAME=placement
DAPR_SENTRY_IMAGE_NAME=sentry
```



```makefile
# build docker image for linux
BIN_PATH=$(OUT_DIR)/$(TARGET_OS)_$(TARGET_ARCH)

ifeq ($(TARGET_OS), windows)
  DOCKERFILE:=Dockerfile-windows
  BIN_PATH := $(BIN_PATH)/release
else ifeq ($(origin DEBUG), undefined)
  DOCKERFILE:=Dockerfile
  BIN_PATH := $(BIN_PATH)/release
else ifeq ($(DEBUG),0)
  DOCKERFILE:=Dockerfile
  BIN_PATH := $(BIN_PATH)/release
else
  DOCKERFILE:=Dockerfile-debug
  BIN_PATH := $(BIN_PATH)/debug
endif

ifeq ($(TARGET_ARCH),arm)
  DOCKER_IMAGE_PLATFORM:=$(TARGET_OS)/arm/v7
else ifeq ($(TARGET_ARCH),arm64)
  DOCKER_IMAGE_PLATFORM:=$(TARGET_OS)/arm64/v8
else
  DOCKER_IMAGE_PLATFORM:=$(TARGET_OS)/amd64
endif
```

支持的 cpu 架构：

```makefile
# Supported docker image architecture
DOCKERMUTI_ARCH=linux-amd64 linux-arm linux-arm64 windows-amd64
```

还有为 target docker-build 和 docker-push 定义的变量：

```makefile
LINUX_BINS_OUT_DIR=$(OUT_DIR)/linux_$(GOARCH)
DOCKER_IMAGE_TAG=$(DAPR_REGISTRY)/$(DAPR_SYSTEM_IMAGE_NAME):$(DAPR_TAG)
DAPR_RUNTIME_DOCKER_IMAGE_TAG=$(DAPR_REGISTRY)/$(DAPR_RUNTIME_IMAGE_NAME):$(DAPR_TAG)
DAPR_PLACEMENT_DOCKER_IMAGE_TAG=$(DAPR_REGISTRY)/$(DAPR_PLACEMENT_IMAGE_NAME):$(DAPR_TAG)
DAPR_SENTRY_DOCKER_IMAGE_TAG=$(DAPR_REGISTRY)/$(DAPR_SENTRY_IMAGE_NAME):$(DAPR_TAG)

ifeq ($(LATEST_RELEASE),true)
DOCKER_IMAGE_LATEST_TAG=$(DAPR_REGISTRY)/$(DAPR_SYSTEM_IMAGE_NAME):$(LATEST_TAG)
DAPR_RUNTIME_DOCKER_IMAGE_LATEST_TAG=$(DAPR_REGISTRY)/$(DAPR_RUNTIME_IMAGE_NAME):$(LATEST_TAG)
DAPR_PLACEMENT_DOCKER_IMAGE_LATEST_TAG=$(DAPR_REGISTRY)/$(DAPR_PLACEMENT_IMAGE_NAME):$(LATEST_TAG)
DAPR_SENTRY_DOCKER_IMAGE_LATEST_TAG=$(DAPR_REGISTRY)/$(DAPR_SENTRY_IMAGE_NAME):$(LATEST_TAG)
endif


# To use buildx: https://github.com/docker/buildx#docker-ce
export DOCKER_CLI_EXPERIMENTAL=enabled
```

### Target: check-docker-env

```makefile
# check the required environment variables
check-docker-env:
ifeq ($(DAPR_REGISTRY),)
	$(error DAPR_REGISTRY environment variable must be set)
endif
ifeq ($(DAPR_TAG),)
	$(error DAPR_TAG environment variable must be set)
endif
```

检查 DAPR_REGISTRY 和 DAPR_TAG 两个环境变量是否设置，验证如下：

```bash
$ make check-docker-env
docker/docker.mk:76: *** DAPR_REGISTRY environment variable must be set.  Stop.
$ export DAPR_REGISTRY=docker.io/skyao
$ make check-docker-env               
docker/docker.mk:79: *** DAPR_TAG environment variable must be set.  Stop.
$ export DAPR_TAG=dev
$ make check-docker-env
make: Nothing to be done for `check-docker-env'.
```

### Target: check-docker-env

```makefile
check-arch:
ifeq ($(TARGET_OS),)
	$(error TARGET_OS environment variable must be set)
endif
ifeq ($(TARGET_ARCH),)
	$(error TARGET_ARCH environment variable must be set)
endif
```

检查 TARGET_OS 和 TARGET_ARCH 两个环境变量是否设置，验证如下：

```bash
$ make check-arch      
make: Nothing to be done for `check-arch'.
```

### Target: docker-build

```makefile
docker-build: check-docker-env check-arch
	$(info Building $(DOCKER_IMAGE_TAG) docker image ...)
ifeq ($(TARGET_ARCH),amd64)
	$(DOCKER) build --build-arg PKG_FILES=* -f $(DOCKERFILE_DIR)/$(DOCKERFILE) $(BIN_PATH) -t $(DOCKER_IMAGE_TAG)-$(TARGET_OS)-$(TARGET_ARCH)
	$(DOCKER) build --build-arg PKG_FILES=daprd -f $(DOCKERFILE_DIR)/$(DOCKERFILE) $(BIN_PATH) -t $(DAPR_RUNTIME_DOCKER_IMAGE_TAG)-$(TARGET_OS)-$(TARGET_ARCH)
	$(DOCKER) build --build-arg PKG_FILES=placement -f $(DOCKERFILE_DIR)/$(DOCKERFILE) $(BIN_PATH) -t $(DAPR_PLACEMENT_DOCKER_IMAGE_TAG)-$(TARGET_OS)-$(TARGET_ARCH)
	$(DOCKER) build --build-arg PKG_FILES=sentry -f $(DOCKERFILE_DIR)/$(DOCKERFILE) $(BIN_PATH) -t $(DAPR_SENTRY_DOCKER_IMAGE_TAG)-$(TARGET_OS)-$(TARGET_ARCH)
else
	-$(DOCKER) buildx create --use --name daprbuild
	-$(DOCKER) run --rm --privileged multiarch/qemu-user-static --reset -p yes
	$(DOCKER) buildx build --build-arg PKG_FILES=* --platform $(DOCKER_IMAGE_PLATFORM) -f $(DOCKERFILE_DIR)/$(DOCKERFILE) $(BIN_PATH) -t $(DOCKER_IMAGE_TAG)-$(TARGET_OS)-$(TARGET_ARCH)
	$(DOCKER) buildx build --build-arg PKG_FILES=daprd --platform $(DOCKER_IMAGE_PLATFORM) -f $(DOCKERFILE_DIR)/$(DOCKERFILE) $(BIN_PATH) -t $(DAPR_RUNTIME_DOCKER_IMAGE_TAG)-$(TARGET_OS)-$(TARGET_ARCH)
	$(DOCKER) buildx build --build-arg PKG_FILES=placement --platform $(DOCKER_IMAGE_PLATFORM) -f $(DOCKERFILE_DIR)/$(DOCKERFILE) $(BIN_PATH) -t $(DAPR_PLACEMENT_DOCKER_IMAGE_TAG)-$(TARGET_OS)-$(TARGET_ARCH)
	$(DOCKER) buildx build --build-arg PKG_FILES=sentry --platform $(DOCKER_IMAGE_PLATFORM) -f $(DOCKERFILE_DIR)/$(DOCKERFILE) $(BIN_PATH) -t $(DAPR_SENTRY_DOCKER_IMAGE_TAG)-$(TARGET_OS)-$(TARGET_ARCH)
endif
```

在执行 docker-build 之前，需要先执行 target build-linux，不然会提示：

```bash
$ make docker-build 
Building docker.io/skyao/dapr:dev docker image ...
docker build --build-arg PKG_FILES=* -f ./docker/Dockerfile ./dist/linux_amd64/release -t docker.io/skyao/dapr:dev-linux-amd64
unable to prepare context: path "./dist/linux_amd64/release" not found
make: *** [docker-build] Error 1
```

#### m1上的构建

 target build-linux 会构建 linux_arm64 版本的二进制文件：

```bash
$ make build-linux 
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_arm64/release/daprd ./cmd/daprd/;
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_arm64/release/placement ./cmd/placement/;
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_arm64/release/operator ./cmd/operator/;
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_arm64/release/injector ./cmd/injector/;
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_arm64/release/sentry ./cmd/sentry/;
```

