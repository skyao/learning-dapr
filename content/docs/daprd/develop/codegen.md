---
title: "Daprd Proto代码生成"
linkTitle: "Proto代码生成"
weight: 2013
date: 2021-03-30
description: >
  Daprd proto代码生成的方法和常见问题
---

{{% pageinfo %}}
Dapr 新版本中对proto文件的代码生成有了极大改进，非常方便。
{{% /pageinfo %}}

Dapr 新版本中proto文件的代码生成可以简单参考官方 README 文件的说明:

https://github.com/dapr/dapr/tree/master/dapr

### 步骤一：安装Protoc

目前 daprd 要求的版本是 [v3.14.0](https://github.com/protocolbuffers/protobuf/releases/tag/v3.14.0)。

#### linux 安装

如果之前安装过其他，则需要删除已经安装的版本：

```bash
sudo rm -rf /usr/local/include/google/protobuf/
sudo rm /usr/local/bin/protoc
```

下载并解压缩之后，按照 readme.txt 文件的提示，复制bin文件和clude目录到合适的位置：

```bash
$ unzip protoc-3.14.0-linux-x86_64.zip

$ sudo cp -r include/google/ /usr/local/include/
# 需要设置权限可读和可执行，755
$ sudo chmod -R 755 /usr/local/include/google
$ sudo cp bin/protoc /usr/local/bin/
$ sudo chmod +x /usr/local/bin/protoc
```

验证安装结果：

```bash
$ protoc --version
libprotoc 3.14.0
```

#### Mac 安装

如果之前安装过其他，则需要删除已经安装的版本：

```bash
sudo rm -rf /usr/local/include/google/protobuf/
sudo rm /usr/local/bin/protoc
```

下载 protoc-3.14.0-osx-x86_64.zip 文件，解压缩之后

```bash
cd protoc-3.14.0-osx-x86_64
sudo cp -r include/google/ /usr/local/include/
# 需要设置权限可读和可执行，755
$ sudo chmod -R 755 /usr/local/include/google
$ sudo cp bin/protoc /usr/local/bin/
$ sudo chmod +x /usr/local/bin/protoc
```

protoc 安装完成之后，验证一下版本：

```bash
$ protoc --version
libprotoc 3.14.0
```

### 步骤二：初始化proto工具

安装 protoc-gen-go 和 protoc-gen-go，dapr的 make file 为此准备了专门的命令 `init-proto`：

```bash
$ make init-proto
go get google.golang.org/protobuf/cmd/protoc-gen-go@v1.25.0 google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.1.0
go: found google.golang.org/protobuf/cmd/protoc-gen-go in google.golang.org/protobuf v1.25.0
```

### 步骤三：从proto生成代码

终于可以开始正式的代码生成了，dapr的 make file 也为此准备了专门的命令 `gen-proto`：

```bash
$ make gen-proto
protoc --go_out=. --go_opt=module=github.com/dapr/dapr --go-grpc_out=. --go-grpc_opt=require_unimplemented_servers=false,module=github.com/dapr/dapr ./dapr/proto/common/v1/*.proto
protoc --go_out=. --go_opt=module=github.com/dapr/dapr --go-grpc_out=. --go-grpc_opt=require_unimplemented_servers=false,module=github.com/dapr/dapr ./dapr/proto/internals/v1/*.proto
protoc --go_out=. --go_opt=module=github.com/dapr/dapr --go-grpc_out=. --go-grpc_opt=require_unimplemented_servers=false,module=github.com/dapr/dapr ./dapr/proto/operator/v1/*.proto
protoc --go_out=. --go_opt=module=github.com/dapr/dapr --go-grpc_out=. --go-grpc_opt=require_unimplemented_servers=false,module=github.com/dapr/dapr ./dapr/proto/placement/v1/*.proto
protoc --go_out=. --go_opt=module=github.com/dapr/dapr --go-grpc_out=. --go-grpc_opt=require_unimplemented_servers=false,module=github.com/dapr/dapr ./dapr/proto/runtime/v1/*.proto
protoc --go_out=. --go_opt=module=github.com/dapr/dapr --go-grpc_out=. --go-grpc_opt=require_unimplemented_servers=false,module=github.com/dapr/dapr ./dapr/proto/sentry/v1/*.proto
go mod tidy
```

### 常见错误

#### 找不到共享文件

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


