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

## Target: check-docker-env

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

## Target: check-docker-env

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

## Target: docker-build

在执行 docker-build 之前，最好先设置好相关的环境变量：

```bash
export DAPR_REGISTRY=docker.io/skyao
export DAPR_TAG=dev			# 这个先别设置，用默认值
export TARGET_OS=linux		# 默认是linux
export TARGET_ARCH=arm64  # 默认是amd64，m1上需要修改
```

然后执行：

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

### m1上的构建

 target build-linux 会构建 linux_arm64 版本的二进制文件：

```bash
$ make build-linux 
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_arm64/release/daprd ./cmd/daprd/;
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_arm64/release/placement ./cmd/placement/;
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_arm64/release/operator ./cmd/operator/;
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_arm64/release/injector ./cmd/injector/;
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.gitcommit=551722f533afa5dfee97482fe3e63d8ff6233d50 -X github.com/dapr/dapr/pkg/version.gitversion=v1.5.1-rc.3-411-g551722f -X github.com/dapr/dapr/pkg/version.version=edge -X github.com/dapr/kit/logger.DaprVersion=edge -s -w" -o ./dist/linux_arm64/release/sentry ./cmd/sentry/;
```

在 m1 上执行 docker-build ：

```bash
$ make docker-build     

Building docker.io/skyao/dapr:dev docker image ...
docker buildx create --use --name daprbuild
error: existing instance for daprbuild but no append mode, specify --node to make changes for existing instances
make: [docker-build] Error 1 (ignored)
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
Unable to find image 'multiarch/qemu-user-static:latest' locally
latest: Pulling from multiarch/qemu-user-static
01c2cdc13739: Pull complete 
11933eee4160: Pull complete 
30abb83a18eb: Pull complete 
0657daef200b: Pull complete 
10094524a9f3: Pull complete 
Digest: sha256:2c8b8fcf1d6badfca797c3fb46b7bb5f705ec7e66363e1cfeb7b7d4c7086e360
Status: Downloaded newer image for multiarch/qemu-user-static:latest
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
Error while loading /qemu-binfmt-conf.sh: Exec format error
make: [docker-build] Error 1 (ignored)
docker buildx build --build-arg PKG_FILES=* --platform linux/arm64/v8 -f ./docker/Dockerfile ./dist/linux_arm64/release -t docker.io/skyao/dapr:dev-linux-arm64
WARN[0000] No output specified for docker-container driver. Build result will only remain in the build cache. To push result image into registry use --push or to load image into docker use --load 
[+] Building 84.8s (12/12) FINISHED                                                                                                                                                                             
 => [internal] booting buildkit                                                                                                                                                                           69.4s
 => => pulling image moby/buildkit:buildx-stable-1                                                                                                                                                        69.0s
 => => creating container buildx_buildkit_daprbuild0                                                                                                                                                       0.4s
 => [internal] load build definition from Dockerfile                                                                                                                                                       0.0s
 => => transferring dockerfile: 297B                                                                                                                                                                       0.0s
 => [internal] load .dockerignore                                                                                                                                                                          0.0s
 => => transferring context: 2B                                                                                                                                                                            0.0s
 => [internal] load metadata for gcr.io/distroless/static:nonroot                                                                                                                                          4.7s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                                                                                           6.9s
 => [auth] library/alpine:pull token for registry-1.docker.io                                                                                                                                              0.0s
 => [alpine 1/2] FROM docker.io/library/alpine:latest@sha256:ceeae2849a425ef1a7e591d8288f1a58cdf1f4e8d9da7510e29ea829e61cf512                                                                              0.2s
 => => resolve docker.io/library/alpine:latest@sha256:ceeae2849a425ef1a7e591d8288f1a58cdf1f4e8d9da7510e29ea829e61cf512                                                                                     0.0s
 => => sha256:a5e44472bb1f0d721d23f82fa10a4c3d25994790238a173c1de950a649eb9a90 2.71MB / 2.71MB                                                                                                             2.9s
 => => extracting sha256:a5e44472bb1f0d721d23f82fa10a4c3d25994790238a173c1de950a649eb9a90                                                                                                                  0.2s
 => [internal] load build context                                                                                                                                                                          7.4s
 => => transferring context: 233.03MB                                                                                                                                                                      7.3s
 => [stage-1 1/4] FROM gcr.io/distroless/static:nonroot@sha256:80c956fb0836a17a565c43a4026c9c80b2013c83bea09f74fa4da195a59b7a99                                                                            0.2s
 => => resolve gcr.io/distroless/static:nonroot@sha256:80c956fb0836a17a565c43a4026c9c80b2013c83bea09f74fa4da195a59b7a99                                                                                    0.0s
 => => sha256:dbcab61d5a5a806aee6156f2e22c601a52119ca8eaeb8fcd08187f22c35d9b88 803.83kB / 803.83kB                                                                                                         3.6s
 => => extracting sha256:dbcab61d5a5a806aee6156f2e22c601a52119ca8eaeb8fcd08187f22c35d9b88                                                                                                                  0.2s
 => [alpine 2/2] RUN apk add -U --no-cache ca-certificates                                                                                                                                                 4.5s
 => [stage-1 2/4] COPY --from=alpine /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/                                                                                                                    0.0s
 => [stage-1 3/4] COPY /* /                                                                                                                                                                                0.2s
docker buildx build --build-arg PKG_FILES=daprd --platform linux/arm64/v8 -f ./docker/Dockerfile ./dist/linux_arm64/release -t docker.io/skyao/daprd:dev-linux-arm64                                            
WARN[0000] No output specified for docker-container driver. Build result will only remain in the build cache. To push result image into registry use --push or to load image into docker use --load             
[+] Building 1.3s (10/10) FINISHED                                                                                                                                                                              
 => [internal] load build definition from Dockerfile                                                                                                                                                       0.0s
 => => transferring dockerfile: 297B                                                                                                                                                                       0.0s
 => [internal] load .dockerignore                                                                                                                                                                          0.0s
 => => transferring context: 2B                                                                                                                                                                            0.0s
 => [internal] load metadata for gcr.io/distroless/static:nonroot                                                                                                                                          0.9s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                                                                                           0.5s
 => [alpine 1/2] FROM docker.io/library/alpine:latest@sha256:ceeae2849a425ef1a7e591d8288f1a58cdf1f4e8d9da7510e29ea829e61cf512                                                                              0.0s
 => => resolve docker.io/library/alpine:latest@sha256:ceeae2849a425ef1a7e591d8288f1a58cdf1f4e8d9da7510e29ea829e61cf512                                                                                     0.0s
 => [internal] load build context                                                                                                                                                                          0.1s
 => => transferring context: 29B                                                                                                                                                                           0.0s
 => [stage-1 1/4] FROM gcr.io/distroless/static:nonroot@sha256:80c956fb0836a17a565c43a4026c9c80b2013c83bea09f74fa4da195a59b7a99                                                                            0.0s
 => => resolve gcr.io/distroless/static:nonroot@sha256:80c956fb0836a17a565c43a4026c9c80b2013c83bea09f74fa4da195a59b7a99                                                                                    0.0s
 => CACHED [alpine 2/2] RUN apk add -U --no-cache ca-certificates                                                                                                                                          0.0s
 => CACHED [stage-1 2/4] COPY --from=alpine /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/                                                                                                             0.0s
 => [stage-1 3/4] COPY /daprd /                                                                                                                                                                            0.1s
docker buildx build --build-arg PKG_FILES=placement --platform linux/arm64/v8 -f ./docker/Dockerfile ./dist/linux_arm64/release -t docker.io/skyao/placement:dev-linux-arm64
WARN[0000] No output specified for docker-container driver. Build result will only remain in the build cache. To push result image into registry use --push or to load image into docker use --load 
[+] Building 1.9s (10/10) FINISHED                                                                                                                                                                              
 => [internal] load build definition from Dockerfile                                                                                                                                                       0.0s
 => => transferring dockerfile: 297B                                                                                                                                                                       0.0s
 => [internal] load .dockerignore                                                                                                                                                                          0.0s
 => => transferring context: 2B                                                                                                                                                                            0.0s
 => [internal] load metadata for gcr.io/distroless/static:nonroot                                                                                                                                          1.2s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                                                                                           0.5s
 => [alpine 1/2] FROM docker.io/library/alpine:latest@sha256:ceeae2849a425ef1a7e591d8288f1a58cdf1f4e8d9da7510e29ea829e61cf512                                                                              0.0s
 => => resolve docker.io/library/alpine:latest@sha256:ceeae2849a425ef1a7e591d8288f1a58cdf1f4e8d9da7510e29ea829e61cf512                                                                                     0.0s
 => [internal] load build context                                                                                                                                                                          0.5s
 => => transferring context: 14.42MB                                                                                                                                                                       0.5s
 => [stage-1 1/4] FROM gcr.io/distroless/static:nonroot@sha256:80c956fb0836a17a565c43a4026c9c80b2013c83bea09f74fa4da195a59b7a99                                                                            0.0s
 => => resolve gcr.io/distroless/static:nonroot@sha256:80c956fb0836a17a565c43a4026c9c80b2013c83bea09f74fa4da195a59b7a99                                                                                    0.0s
 => CACHED [alpine 2/2] RUN apk add -U --no-cache ca-certificates                                                                                                                                          0.0s
 => CACHED [stage-1 2/4] COPY --from=alpine /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/                                                                                                             0.0s
 => [stage-1 3/4] COPY /placement /                                                                                                                                                                        0.0s
docker buildx build --build-arg PKG_FILES=sentry --platform linux/arm64/v8 -f ./docker/Dockerfile ./dist/linux_arm64/release -t docker.io/skyao/sentry:dev-linux-arm64
WARN[0000] No output specified for docker-container driver. Build result will only remain in the build cache. To push result image into registry use --push or to load image into docker use --load 
[+] Building 2.5s (10/10) FINISHED                                                                                                                                                                              
 => [internal] load build definition from Dockerfile                                                                                                                                                       0.0s
 => => transferring dockerfile: 297B                                                                                                                                                                       0.0s
 => [internal] load .dockerignore                                                                                                                                                                          0.0s
 => => transferring context: 2B                                                                                                                                                                            0.0s
 => [internal] load metadata for gcr.io/distroless/static:nonroot                                                                                                                                          1.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                                                                                           0.5s
 => [alpine 1/2] FROM docker.io/library/alpine:latest@sha256:ceeae2849a425ef1a7e591d8288f1a58cdf1f4e8d9da7510e29ea829e61cf512                                                                              0.0s
 => => resolve docker.io/library/alpine:latest@sha256:ceeae2849a425ef1a7e591d8288f1a58cdf1f4e8d9da7510e29ea829e61cf512                                                                                     0.0s
 => [internal] load build context                                                                                                                                                                          1.3s
 => => transferring context: 36.64MB                                                                                                                                                                       1.3s
 => [stage-1 1/4] FROM gcr.io/distroless/static:nonroot@sha256:80c956fb0836a17a565c43a4026c9c80b2013c83bea09f74fa4da195a59b7a99                                                                            0.0s
 => => resolve gcr.io/distroless/static:nonroot@sha256:80c956fb0836a17a565c43a4026c9c80b2013c83bea09f74fa4da195a59b7a99                                                                                    0.0s
 => CACHED [alpine 2/2] RUN apk add -U --no-cache ca-certificates                                                                                                                                          0.0s
 => CACHED [stage-1 2/4] COPY --from=alpine /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/                                                                                                             0.0s
 => [stage-1 3/4] COPY /sentry /        
```

### amd64 上的构建

TBD

