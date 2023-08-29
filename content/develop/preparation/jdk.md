---
title: "安装jdk/maven等"
linkTitle: "Java"
weight: 21
date: 2022-01-24
description: >
  [可选]安装jdk/maven等Java开发工具
---



如果需要开发 dapr java-sdk，则需要安装 Java SDK。

为了方便在多个 JDK 版本中切换，建议采用 sdkman 来管理 JDK 版本。

## 安装 sdkman

参考：

- https://skyao.io/learning-ubuntu-server/docs/development/common/sdkman.html
- https://skyao.io/learning-macos/docs/programing/common/sdkman.html

## 判断需要的 jdk 版本

参考java sdk 项目的 maven pom 文件设定：

https://github.com/dapr/java-sdk/blob/master/pom.xml

对 java source 和 target 的要求都是8，也就是 Java 8

```bash
    <maven.compiler.source>8</maven.compiler.source>
    <maven.compiler.target>8</maven.compiler.target>
```

## 安装 jdk 版本

TBD

```bash
sdk list java
sdk install java 11.0.20-zulu
```


## 安装 mvn 