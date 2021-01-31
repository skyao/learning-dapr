---
title: "安全的概念"
linkTitle: "概念"
weight: 1002
date: 2021-01-31
description: >
  Dapr的安全的概念
---


> 内容节选自：https://github.com/dapr/docs/blob/master/concepts/secrets/



### Dapr秘密管理

Dapr为开发人员提供了一种一致的方式来提取应用 secrets，而无需了解所使用的 secrets store 的详细信息。 secrets store 是Dapr中的组件。Dapr允许用户编写新的  secrets store  组件实现，这些实现既可以用来保存其他Dapr组件的 secrets（例如，状态存储组件用来读取/写入状态的secrets），也可以为应用提供专用的 secrets 构建块API。使用 secrets 构造块API，您可以轻松地从命名的 secrets store 读取应用可以使用的 secrets。

Secrets store 的例子包括`Kubernetes`，`Hashicorp Vault`，`Azure KeyVault`。

### 在Dapr Components中引用 secrets store

您可以将凭据放在 Dapr 支持的 secrets store 中，并在 Dapr 组件中引用该 secrets，而不是在Dapr组件中包括凭据。

> 学习心得:
>
> https://github.com/dapr/docs/blob/master/howto/setup-secret-store/gcp-secret-manager.md
>
> 这个文档的最下面的用法，我觉得还是挺有参考价值的。 sidecar 搞定 secrets 的数据，放在自己内容，其他模块可直接引用，应用根本不需要真正拿到这些敏感信息。比如这个例子里面，应用连接 redis 去做 状态管理，redis 的连接密码就这么轻松搞定
> 
> 下沉的能力越多，build block越多，类似的需要做安全管理的字段越多，这个玩法的价值就越大



### 检索Secrets

服务代码可以调用 secrets 构造块API，以从Dapr支持的 secret store 中检索 secrets。



