---
title: "Dapr的构建"
linkTitle: "构建"
weight: 111
date: 2021-01-29
description: >
  Dapr的构建
---

## build

在项目根目录下执行 ：

```bash
$ make build
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.commit=v0.8.0-rc.2-96-g79a1f14 -X github.com/dapr/dapr/pkg/version.version=edge -s -w" -o ./dist/darwin_amd64/release/daprd ./cmd/daprd/main.go;
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.commit=v0.8.0-rc.2-96-g79a1f14 -X github.com/dapr/dapr/pkg/version.version=edge -s -w" -o ./dist/darwin_amd64/release/placement ./cmd/placement/main.go;
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.commit=v0.8.0-rc.2-96-g79a1f14 -X github.com/dapr/dapr/pkg/version.version=edge -s -w" -o ./dist/darwin_amd64/release/operator ./cmd/operator/main.go;
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.commit=v0.8.0-rc.2-96-g79a1f14 -X github.com/dapr/dapr/pkg/version.version=edge -s -w" -o ./dist/darwin_amd64/release/injector ./cmd/injector/main.go;
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build  -ldflags="-X github.com/dapr/dapr/pkg/version.commit=v0.8.0-rc.2-96-g79a1f14 -X github.com/dapr/dapr/pkg/version.version=edge -s -w" -o ./dist/darwin_amd64/release/sentry ./cmd/sentry/main.go;

```

构建完成之后得到的文件：

```bash
$ ls -lh ./dist/darwin_amd64/release/
total 429128
-rwxr-xr-x  1 aoxiaojian  staff    89M Aug 13 17:19 daprd
-rwxr-xr-x  1 aoxiaojian  staff    33M Aug 13 17:20 injector
-rwxr-xr-x  1 aoxiaojian  staff    35M Aug 13 17:20 operator
-rwxr-xr-x  1 aoxiaojian  staff    12M Aug 13 17:19 placement
-rwxr-xr-x  1 aoxiaojian  staff    33M Aug 13 17:20 sentry
```







