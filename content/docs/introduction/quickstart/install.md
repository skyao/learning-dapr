---
title: "安装 dapr"
linkTitle: "安装"
weight: 10
date: 2022-04-09
description: >
  安装Dapr并进行初始化
---

## 安装 dapr

### 安装 dapr CLI

TBD

### 初始化 dapr

此时如果直接用 dapr run 命令启动应用和 sidecar，会报错：

```bash
$ dapr run --app-id invokedemo --app-port 3000 -- java -jar target/dapr-java-sdk-examples-exec.jar io.dapr.examples.invoke.http.DemoService -p 3000
ℹ️  Starting Dapr with id invokedemo. HTTP Port: 42127. gRPC Port: 44623
❌  fork/exec /home/sky/.dapr/bin/daprd: no such file or directory
```

这是因为 daprd 等二进制文件还没有安装，需要执行 `dapr init ` 先初始化安装 dapr：

```bash
$ dapr init                                                                                                                                        
⌛  Making the jump to hyperspace...
ℹ️  Installing runtime version 1.6.0
❌  Downloading binaries and setting up components...
❌  error downloading dashboard binary: Get "https://github.com/dapr/dashboard/releases/download/v0.9.0/dashboard_linux_amd64.tar.gz": unexpected EOF
```

很不幸的是这个命令是直接从 github 网站下载，经常出现问题，无法连接或者下载中途中断。最好是先把科学上网的代理设置好，再执行：

```bash
$ dapr init      
⌛  Making the jump to hyperspace...
ℹ️  Installing runtime version 1.6.0
→  Downloading binaries and setting up components... 
Dapr runtime installed to /home/sky/.dapr/bin, you may run the following to add it to your path if you want to run daprd directly:
    export PATH=$PATH:/home/sky/.dapr/bin                                                                         ✅  Downloading binaries and setting up components...
✅  Downloaded binaries and completed components set up.
ℹ️  daprd binary has been installed to /home/sky/.dapr/bin.
ℹ️  dapr_placement container is running.
ℹ️  dapr_redis container is running.
ℹ️  dapr_zipkin container is running.
ℹ️  Use `docker ps` to check running containers.
✅  Success! Dapr is up and running. To get started, go here: https://aka.ms/dapr-getting-started
```



