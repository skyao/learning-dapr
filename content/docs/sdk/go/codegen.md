---
title: "Go SDK的proto代码生成"
linkTitle: "proto代码生成"
weight: 3102
date: 2021-02-08
description: >
  Dapr的Go SDK中从protos生成代码
---

## 准备工作

### 准备protoc

安装Protoc，目前 daprd 要求的版本是 [v3.14.0](https://github.com/protocolbuffers/protobuf/releases/tag/v3.14.0)。

具体见 [Daprd Proto代码生成]( {{< relref "/docs/daprd/develop/codegen.md" >}})

sdk 和 daprd保持一致，但实际目前go sdk采用的是 protoc 3.11.2。

## 将proto生成源码

现在 go sdk的make file 提供了 protos 命令来从proto生成go代码：

```bash
$ make protos
# 下载安装gogoreplace
go install github.com/gogo/protobuf/gogoreplace
go: finding module for package github.com/gogo/protobuf/gogoreplace
go: found github.com/gogo/protobuf/gogoreplace in github.com/gogo/protobuf v1.3.2
# 删除本地的proto文件
rm -f ./dapr/proto/common/v1/*
rm -f ./dapr/proto/runtime/v1/*
# 从 dapr/dapr 仓库下载 common.proto 文件到本地
wget -q https://raw.githubusercontent.com/dapr/dapr/master/dapr/proto//common/v1/common.proto -O ./dapr/proto/common/v1/common.proto
# 使用 gogoreplace 工具替换 common.proto 文件中的 go_package
gogoreplace 'option go_package = "github.com/dapr/dapr/pkg/proto/common/v1;common";' \
        'option go_package = "github.com/dapr/go-sdk/dapr/proto/common/v1;common";' \
        ./dapr/proto/common/v1/common.proto
wget -q https://raw.githubusercontent.com/dapr/dapr/master/dapr/proto//runtime/v1/appcallback.proto -O ./dapr/proto/runtime/v1/appcallback.proto
gogoreplace 'option go_package = "github.com/dapr/dapr/pkg/proto/runtime/v1;runtime";' \
        'option go_package = "github.com/dapr/go-sdk/dapr/proto/runtime/v1;runtime";' \
        ./dapr/proto/runtime/v1/appcallback.proto
wget -q https://raw.githubusercontent.com/dapr/dapr/master/dapr/proto//runtime/v1/dapr.proto -O ./dapr/proto/runtime/v1/dapr.proto
gogoreplace 'option go_package = "github.com/dapr/dapr/pkg/proto/runtime/v1;runtime";' \
        'option go_package = "github.com/dapr/go-sdk/dapr/proto/runtime/v1;runtime";' \
        ./dapr/proto/runtime/v1/dapr.proto
# 从proto文件生成go代码
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
           dapr/proto/common/v1/common.proto
protoc --go_out=. --go_opt=paths=source_relative \
           --go-grpc_out=. --go-grpc_opt=paths=source_relative \
           dapr/proto/runtime/v1/*.proto
# 删除刚才下载到本地的proto
rm -f ./dapr/proto/common/v1/*.proto
rm -f ./dapr/proto/runtime/v1/*.proto
```

## 后记：更新protoc版本

发现 go sdk 使用的protoc 版本是 v3.11.2，而之前我为了满足 dapr/dapr 仓库的要求安装的是 protoc v3.14.0。两个版本生成的代码会有一些细微的差异，也就造成了生成的代码会合git 仓库中的现有代码不同。

在不同仓库之间切换使用不同的 protoc 版本实在是太不方便了，最简单的方法还是统一到同一个版本。

提交了issue和PR 给到社区，希望可以升级 go sdk 的 protoc 版本到 v3.14.0：

https://github.com/dapr/go-sdk/pull/141