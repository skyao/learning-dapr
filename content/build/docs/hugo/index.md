---
title: "安装配置hugo"
linkTitle: "安装hugo"
weight: 2
date: 2022-03-10
description: >
  准备 hugo 以便构建文档
---



## 准备工作

### 安装golang

### 安装 node

ubuntu下：

```bash
wget https://nodejs.org/dist/v18.18.0/node-v18.18.0-linux-x64.tar.xz
tar xvf node-v18.18.0-linux-x64.tar.xz
sudo mv node-v18.18.0-linux-x64 /usr/share/nodejs
```

然后修改

```bash
vi ~/.zshrc
```

加入:

```bash
export PATH=/usr/share/nodejs/bin:$PATH
```

生效并检查：

```bash
source ~/.zshrc
npm --version
```

## 安装hugo

```bash
wget https://github.com/gohugoio/hugo/releases/download/v0.119.0/hugo_extended_0.119.0_linux-amd64.deb
sudo dpkg -i hugo_extended_0.119.0_linux-amd64.deb
```



