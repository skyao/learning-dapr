---
title: "背景"
linkTitle: "背景"
weight: 10
date: 2021-02-24
description: >
  java sdk 中序列化的背景
---

### 文档介绍

https://github.com/dapr/java-sdk#how-to-use-a-custom-serializer

dapr java-sdk 项目的 readme 中有这么一段介绍：

> How to use a custom serializer

如何使用一个自定义的序列化器

> This SDK provides a basic serialization for request/response objects but also for state objects. Applications should provide their own serialization for production scenarios.

这个SDK为请求/响应对象提供了一个基本的序列化，但也为状态对象提供了序列化。应用程序应该为生产场景提供他们自己的序列化。

