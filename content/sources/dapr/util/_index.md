---
type: docs
title: "工具类代码的源码学习"
linkTitle: "工具类代码"
weight: 100
date: 2021-03-03
description: >
  Dapr 工具类代码的源码学习
---

工具类代码指完全作为工具使用的代码，这些代码往往是在代码调用链的最底层，自身没有任何特定逻辑，只专注于完成某个特定的功能，作为上层代码的工具使用。

工具类代码处于代码依赖关系的最底层。