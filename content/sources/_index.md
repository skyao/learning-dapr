---
type: docs
title: "Dapr源码学习"
linkTitle: "源码"
weight: 1
date: 2021-02-22
cascade:
- type: "docs"
menu:
  main:
    weight: 30
    pre: <i class='fa fa-code'></i>
---

将按照两个维度进行 dapr 源码学习：

1. 调用流程

   根据每个构建块提供的功能，分析从请求发出到请求处理完成的整个调用流程。

   主要目标是了解请求处理的主流程和代码实现方式，以及相关的结构设计，不深入展开细节。

2. 代码仓库

   按照每个代码仓库来遍历所有代码实现，会展开所有细节。

   主要目标是摸清dapr代码实现的每一个角落，实现对代码的全面了解。

{{% pageinfo %}}
目标：深入学习 Dapr 源代码，深度掌握 dapr 设计实现
{{% /pageinfo %}}