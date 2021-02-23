---
type: blog
date: 2021-02-19
title: "支持自定义k8s secret存储组件"
linkTitle: "支持自定义k8s secret存储组件"
description: >
  组件模型的主要好处之一是代码和基础设施的解耦。在大多数情况下，代码与提供构建块能力的组件是不可知的。当使用Kubernetes secret 存储时，情况并非如此
---

组件模型的主要好处之一是代码和基础设施的解耦。在大多数情况下，代码与提供构建块能力的组件是不可知的。当使用Kubernetes secret 存储时，情况并非如此。

## Proposal信息

[Support for custom Kubernetes secret store component #2821](https://github.com/dapr/dapr/issues/2821)

### 提案内容

One of the key benefits of the component model is the decoupling of code and infrastructure. In most cases, code is agnostic to the component providing the building block capability. This is not the case when using the Kubernetes secret store where the store name is `kubernetes` and a component YAML is not used to define it (at least [component documentation](https://docs.dapr.io/operations/components/setup-secret-store/supported-secret-stores/kubernetes-secret-store/) does not indicate how to do so)

> 组件模型的主要好处之一是代码和基础设施的解耦。在大多数情况下，代码与提供构建块能力的组件是不可知的。当使用Kubernetes secret 存储时，情况并非如此，此时存储名称是 `kubernetes`，并且没有使用组件YAML来定义它（至少组件文档没有说明如何这样做）
>

In addition, generic get of a secret via the secret API requires secret store name + key. In Kubernetes secret store, secrets have a secret name ontop of a key and so the API in this case returns a dictionary `{key: somekey, value: secret}` and so uniform handling of fetched secret value cannot be performed.

> 另外，通过secret API通用获取secret需要secret 存储的 name+key。在 Kubernetes secret 存储中，secret 在key之上有一个secret name，所以这种情况下API会返回一个字典 `{key：somekey，value：secret}`，所以无法对获取的 secret 值进行统一处理。

**Scenario**:

1. Developer is writing and app that gets a secret from a secret store and in local environment uses env var or file secret store in development phase.
2. When deploying to production on k8s, instead of just swapping component files, the developer needs to ensure the code is aware it is running on k8s and use the fixed `kuberentes` secret store name as well as handle a dictionary output

> 场景: 
>
> 1. 开发者正在编写一个从secret存储中获取secret的app，在本地环境下，开发阶段使用环境变量或文件secret存储。
> 2. 部署到生上的 k8s 时，开发者需要确保代码知道它是运行在k8s上，并使用固定的 kuberentes secret 存储名称以及处理字典输出，而不是仅仅交换组件文件。

**Proposal**:

1. Allow defining a component for the kubernetes secret store that lets user define the name
2. In the component metadata allow defining the secret name so the toegther with the key provided with the API will return the secret alone

> 建议：
>
> 1. 允许为 kubernetes secret 存储定义一个组件，让用户定义名称。
> 2. 在组件元数据中，允许定义 secret 名称，以便与API提供的key一起单独返回secret。

One more note - it seems that for the use of secrets in components, the required data includes both secret `name` and `key`.

> 还有一点需要注意--似乎在组件中使用 secret 时，所需数据包括secret `name`和`key`。

Example, for the redis password stored in the Kuberentes secret store, as described [here](https://docs.dapr.io/operations/components/component-secrets/#referencing-a-kubernetes-secret). So that should also be available for getting secrets using the API

> 例如，对于存储在Kuberentes secret 存储中的 redis 密码，如这里所述。因此，这也应该可以用于使用API获取 secret。

This means that the inconsistency also applies to the use of secrets in components:

> 这意味着，不一致的情况也适用于组件中 secret 的使用。

When the secret store is a Kubernetes secret store, reference to a secret in a component looks like this:

> 当 secret 存储是 Kubernetes secret 存储时，组件中对 secret 的引用是这样的：

```yaml
secretKeyRef:
      name: <name of secret>
      key: <key to get secret value>
```

While if the secret store is not a Kubernetes secret store, reference to a secret is:

> 而如果 secret 存储不是 Kubernetes secret 存储时，对 secret 的引用是：

```yaml
secretKeyRef:
      name: <name of secret>
```

### 提案讨论

I agree. We should prioritize fix in scenarios like this where user code changes because of the component of choice.

> 我同意。我们应该在这种因为选择组件而导致用户代码变化的场景下优先修复。

Regarding backwards compatibility - Change does not have to be breaking: if no component file is there, behavior can be as today but if a component is defined for Kubernetes secret store it will override the behavior

> 关于向后兼容性——变更不一定带来破坏：如果没有组件文件，行为可以像现在一样，但如果为 Kubernetes secret 存储定义了一个组件，它将覆盖行为。

