---
title: "安装Protoc"
linkTitle: "Protoc"
weight: 100
date: 2022-10-29
description: >
  安装Protoc用于从proto文件生成代码
---

{{% pageinfo %}}
Dapr 新版本中对proto文件的代码生成有了极大改进，非常方便。
{{% /pageinfo %}}

Dapr 新版本中proto文件的代码生成可以简单参考官方 README 文件的说明:

https://github.com/dapr/dapr/tree/master/dapr

## 步骤一：安装Protoc

目前 daprd 要求的版本是 [[v3.21.12](https://github.com/protocolbuffers/protobuf/releases/tag/v21.12) ](https://github.com/protocolbuffers/protobuf/releases/tag/v21.12)。

### linux-amd64 安装

如果之前安装过其他，则需要删除已经安装的版本：

```bash
sudo rm -rf /usr/local/include/google/protobuf/
sudo rm /usr/local/bin/protoc
```

下载并解压缩之后，按照 readme.txt 文件的提示，复制bin文件和clude目录到合适的位置：

```bash
$ wget https://github.com/protocolbuffers/protobuf/releases/download/v21.12/protoc-21.12-linux-x86_64.zip
$ unzip protoc-21.12-linux-x86_64.zip

$ sudo cp -r include/google/ /usr/local/include/
# 需要设置权限可读和可执行，755
$ sudo chmod -R 755 /usr/local/include/google
$ sudo cp bin/protoc /usr/local/bin/
$ sudo chmod +x /usr/local/bin/protoc
```

验证安装结果：

```bash
$ protoc --version
libprotoc 3.21.12
```

### Macos-amd64 安装

> 备注：不再更新，我已经没有intel cpu的macos了。

### Macos-arm64 安装

如果之前安装过其他，则需要删除已经安装的版本：

```bash
sudo rm -rf /usr/local/include/google/protobuf/
sudo rm /usr/local/bin/protoc
```

下载 pr

```bash
$ wget https://github.com/protocolbuffers/protobuf/releases/download/v21.12/protoc-21.12-osx-aarch_64.zip
$ unzip protoc-21.12-osx-aarch_64.zip
$ sudo cp -r include/ /usr/local/include/
# 需要设置权限可读和可执行，755
$ sudo chmod -R 755 /usr/local/include/google
$ sudo cp bin/protoc /usr/local/bin/
$ sudo chmod +x /usr/local/bin/protoc
```

如果遇到macos禁止protoc运行，在设置中找到 "security & Privacy"，会有 protoc 运行的提示，点击容许即可。

protoc 安装完成之后，验证一下版本：

```bash
$ protoc --version
libprotoc 3.21.12
```



## 步骤二：初始化proto工具

安装 protoc-gen-go 和 protoc-gen-go，dapr的 make file 为此准备了专门的命令 `init-proto`：

```bash
# 进入 dapr/dapr 仓库
$ cd dapr 
$ make init-proto
go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28.1
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2.0
```

## 步骤三：从proto生成代码

终于可以开始正式的代码生成了，dapr的 make file 也为此准备了专门的命令 `gen-proto`：

```bash
# 进入 dapr/dapr 仓库
$ cd dapr 
$ make gen-proto
protoc --go_out=. --go_opt=module=github.com/dapr/dapr --go-grpc_out=. --go-grpc_opt=require_unimplemented_servers=false,module=github.com/dapr/dapr ./dapr/proto/common/v1/*.proto
protoc --go_out=. --go_opt=module=github.com/dapr/dapr --go-grpc_out=. --go-grpc_opt=require_unimplemented_servers=false,module=github.com/dapr/dapr ./dapr/proto/internals/v1/*.proto
protoc --go_out=. --go_opt=module=github.com/dapr/dapr --go-grpc_out=. --go-grpc_opt=require_unimplemented_servers=false,module=github.com/dapr/dapr ./dapr/proto/operator/v1/*.proto
protoc --go_out=. --go_opt=module=github.com/dapr/dapr --go-grpc_out=. --go-grpc_opt=require_unimplemented_servers=false,module=github.com/dapr/dapr ./dapr/proto/placement/v1/*.proto
protoc --go_out=. --go_opt=module=github.com/dapr/dapr --go-grpc_out=. --go-grpc_opt=require_unimplemented_servers=false,module=github.com/dapr/dapr ./dapr/proto/runtime/v1/*.proto
protoc --go_out=. --go_opt=module=github.com/dapr/dapr --go-grpc_out=. --go-grpc_opt=require_unimplemented_servers=false,module=github.com/dapr/dapr ./dapr/proto/sentry/v1/*.proto
go mod tidy
```

执行 `git status` 命令，对比一下新生成的代码和dapr 仓库中已经保存的代码，如果代码没有改动说明我们的protoc代码生成和dapr项目保持一致了。

## 常见错误

### 找不到共享文件

报错如下：

```bash
$ make gen-proto

protoc --go_out=. --go_opt=module=github.com/dapr/dapr --go-grpc_out=. --go-grpc_opt=require_unimplemented_servers=false,module=github.com/dapr/dapr ./dapr/proto/common/v1/*.proto
google/protobuf/any.proto: File not found.
dapr/proto/common/v1/common.proto:10:1: Import "google/protobuf/any.proto" was not found or had errors.
dapr/proto/common/v1/common.proto:55:3: "google.protobuf.Any" is not defined.
dapr/proto/common/v1/common.proto:76:3: "google.protobuf.Any" is not defined.
make: *** [Makefile:260：gen-proto-common] 错误 1
```

这个错误发生的原因通常是 protoc 安装时的 include 文件未能放置好，或者是相关的目录没有权限。经测试验证需要读和可执行权限，因此设置755：

```bash
# 需要设置权限可读和可执行，755
$ sudo chmod -R 755 /usr/local/include/google
```

