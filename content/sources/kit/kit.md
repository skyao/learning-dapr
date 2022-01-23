---
title: "kit仓库简介"
linkTitle: "简介"
weight: 1
date: 2022-01-21
description: >
  存放共享的工具代码
---

### kit仓库的介绍

Shared utility code for Dapr runtime.

https://github.com/dapr/kit

目前内容很少，只有 logger/config/retry 三个package。

### kit仓库的背景

kit 仓库是后来提取出来的仓库，原来的代码存放在 dapr 仓库中，被 dapr 仓库中的其他代码使用。后来 components-contrib 仓库的代码也使用了这些基础代码，这导致了一个循环依赖：

- dapr 仓库依赖 components-contrib 仓库: 使用 components-contrib 仓库 仓库中的各种 components 实现
- components-contrib 仓库依赖dapr 仓库： 使用dapr 仓库中的基础代码。

```plantuml
participant dapr
participant       "components-contrib"       as components
dapr -> components : for component impl 
components -> dapr : for common code
```

为了让依赖关系更加的清晰，避免循环依赖，因此将这些基础代码从 dapr 仓库中移出来存放在单独的 kit仓库中，之后的依赖关系就是这样：

- dapr 仓库依赖 components-contrib 仓库: 使用 components-contrib 仓库 仓库中的各种 components 实现
- dapr 仓库依赖 kit 仓库： 使用 kit 仓库中的基础代码。
- components-contrib 仓库依赖 kit 仓库： 使用 kit 仓库中的基础代码。

```plantuml
participant dapr
participant       "components-contrib"       as components
participant kit

dapr -> kit : for common code
components -> kit : for common code
dapr -> components : for component impl 
```