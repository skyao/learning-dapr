---
title: "安装.net"
linkTitle: ".net"
weight: 40
date: 2021-01-29
description: >
  为运行基于 .net 的 quickstart 做准备
---



## 安装 .net

docker 的 quickstart 对 .net 的要求是

- [.NET SDK or .NET 6 SDK installed](https://dotnet.microsoft.com/download).

实际测试不能用 .NET 7, 只能用 .NET 6

### ubuntu 20.04

参考： https://learn.microsoft.com/en-us/dotnet/core/install/linux-ubuntu-2004

添加仓库：

```bash
wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb
sudo apt-get update
```

安装 dotnet-sdk-6.0 ：

```bash
sudo apt-get install -y dotnet-sdk-6.0
```

安装 aspnetcore-runtime-6.0：

```bash
sudo apt-get install -y aspnetcore-runtime-6.0
```

