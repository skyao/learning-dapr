---
title: "构建daprdocs的hugo文档"
linkTitle: "构建daprdocs"
weight: 100
date: 2022-03-10
description: >
  构建 daprdocs 的hugo文档
---



## 准备工作

### 更新 git submodules

```bash
cd ./daprdocs
git submodule update --init --recursive
```

### 安装 npm 包


```bash
npm install
```

## 构建daprdocs项目

在本机构建：

```bash
hugo server --disableFastRender
```

然后访问 http://localhost:1313/

如果不是在本机构建，则需要修改 bind 地址：

```bash
hugo server --baseURL https://192.168.99.100:1313 --bind="0.0.0.0" --disableFastRender
```

然后通过 http://192.168.99.100:1313/ 进行访问
