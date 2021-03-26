---
type: blog
date: 2021-03-26
title: "Configuration API"
linkTitle: "Configuration API"
author: Mark Fussell ([@msfussell](https://github.com/msfussell))
description: >
  添加 building block 提供 configuration API 
---

添加 building block 提供 configuration API 

## Proposal信息

[Configuration API Building Block #2941](https://github.com/dapr/dapr/issues/2941)

This is a building blocks proposal to include a configuration API for reading and writing application configuration data, for example from Azure Configuration Manager or GCP Configuration Management.

> 这是一个构件提案，旨在包含一个用于读写应用配置数据的配置 API，例如来自 Azure 配置管理器或 GCP 配置管理。

https://docs.microsoft.com/en-us/azure/azure-app-configuration/overview and
https://cloud.google.com/solutions/configuration-management/
https://aws.amazon.com/config/

By abstracting configuration API, this provide portability and pluggability of code.

> 通过抽象配置API，提供代码的可移植性和可插拔性。

This could also be used for local and developer framework configurations abstractions , for example reading data from a local file or integration with ASP.NET Core configuration.

> 这也可以用于本地和开发者框架的配置抽象，例如从本地文件中读取数据或与ASP.NET Core配置集成。

## 提案讨论

#### Jjcollinge

Just an off the cuff thought. I've seen issues binding to cloud based config managers as when they become unavailable (due to an outage in AAD for example 👀) you cannot start your app at all. I wonder if Dapr could provide a fall-through mechanism to support falling back from a primary config provider to an alternative to help this scenario for those who deem it necessary.

> 只是一个脱口而出的想法。我已经看到了与基于云的配置管理器绑定的问题，因为当他们变得不可用时（例如由于AAD的中断👀），你根本无法启动你的应用程序。我想知道Dapr是否可以提供一个回落(fall-through)机制，支持从一个主要的配置提供者回落到一个替代的配置提供者，以帮助那些认为有必要的人解决这种情况。

#### KaiWalter

Requirements I see currently (w/o checking Azure / AWS / GCP capabilities beforehand):
> 我目前看到的v需求（事先没有检查Azure / AWS / GCP能力）。

API：

- GET to configuration scope - single Dapr service (=AppId) and an "application context" covering multiple semantically bounded services

- GET single configuration value

- GET configuration structure (collection of all values) for a scope (see above)

- PUSH (like pub/sub) when single configuration value is changed

> - GET到配置范围--单个Dapr服务(=AppId)和涵盖多个语义绑定服务的 "应用上下文"。
> - GET单个配置值
> - 获取作用域的配置结构（所有值的集合）（见上文）。
> - 当单个配置值被改变时，进行PUSH（像pub/sub）。

*sidecar*

- once configuration values are pushed, those are cached and returned also in cache the underlying service is not available
- cache gets invalidated updated when configuration values change in underlying service

> - 一旦配置值被推送，这些配置值就会被缓存，并在缓存中返回，而底层服务则不可用。
> - 当底层服务的配置值发生变化时，缓存会被失效更新。

*component definition*

- be able to set defaults for configuration values in cache underlying configuration service is not available
> - 能够为缓存中的配置值设置默认值，底层配置服务不可用。

Generally, if shaped right this could even used for feature switching.

> 一般来说，如果实现正确，这甚至可以用于功能切换。

#### msfussell

Yes, this is a real issue when using centralized configuration stores and something that we have experienced in Azure. What you do not want is the app inability to start due to a configuration service being unavailable. You can say the same is also true for secrets.

> 是的，在使用集中式配置存储时，这是一个真实的问题，也是我们在 Azure 中经历过的问题。你不希望看到的是由于配置服务不可用而导致应用无法启动。你可以说对于秘密也是如此。

#### withinboredom

Where will these be cached at? Do we need a cache building block or will Dapr act as a distributed cache?

> 这些东西将被缓存在哪里？我们需要一个缓存构造件还是Dapr将作为一个分布式缓存？

#### KaiWalter

implying that configuration values generally should not consume much space/memory - I'd like to have these cached as close as possible : in Dapr sidecar

> 考虑到配置值一般不应该消耗太多的空间/内存--我希望这些配置值能尽可能地被缓存起来：在Dapr sidecar中。