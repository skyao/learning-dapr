---
type: blog
date: 2020-03-30
title: "状态存储过滤和查询API"
linkTitle: "状态存储过滤和查询API"
description: >
  状态存储过滤和查询API
---

状态存储过滤和查询API

## Proposal信息

[[Proposal] API for state store filtering and querying [like OData] #1339](https://github.com/dapr/dapr/issues/1339)

我们想通过SDK（对我们来说是.NET SDK）中的Dapr.Client 将状态存储作为主存储使用，但是状态存储API只允许对一个存储的文档进行单一的GET请求。

如果能像 Cosmos DB 中描述的 OData 一样，有机会通过查询来检索更多的值，那将是非常好的。



