---
title: "状态管理的参考文档"
linkTitle: "参考文档"
weight: 813
date: 2021-01-31
description: >
  Dapr状态管理的参考文档
---


> 原文：https://github.com/dapr/docs/blob/master/reference/api/state_api.md

Dapr 提供了一个可靠的状态端点，允许开发人员通过API保存和检索状态。Dapr具有可插拔架构，并允许绑定到多个云/内部状态存储。

### component文件

Dapr状态存储组件yaml文件的结构如下：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: <NAME>
  namespace: <NAMESPACE>
spec:
  type: state.<TYPE>
  metadata:
  - name:<KEY>
    value:<VALUE>
  - name: <KEY>
    value: <VALUE>
```

`metadata.name` 是状态存储的名称。

`spec/metadata` 部分是一个开放的键值对元数据，允许绑定定义连接属性。

从0.4.0版本开始，增加了对多个状态存储的支持。这与之前的版本相比是一个breaking change / 不兼容变化，因为状态API是为了支持这个新场景而改变的。

参见  https://github.com/dapr/dapr/blob/master/docs/decision_records/api/API-008-multi-state-store-api-design.md 

### key scheme

Dapr状态存储是键/值存储。为了确保数据兼容性，Dapr要求这些数据存储遵循固定的 key scheme。对于一般状态，key 的格式为：

```
<App ID>||<state key>
```

对于Actor状态，密钥格式为：

```
<App ID>||<Actor type>||<Actor id>||<state key>
```

### Save/Get/Get bulk/Delete状态

见原文。

### 状态事务

将对状态存储的更改作为一个multi-item事务保留下来。

请注意，此操作依赖于使用支持multi-item事务的状态存储组件。

支持事务的状态存储列表。

- Redis
- MongoDB
- PostgreSQL
- SQL Server
- Azure CosmosDB

细节看原文

## Configuring state store for actors

见原文

### 可选行为

### 并发

Dapr使用乐观并发控制（OCC）与ETags。Dapr对状态存储的以下要求是可选的。

- 与Dapr兼容的状态存储可以使用ETag支持乐观并发控制。当ETag与save 或 delete 请求相关联时，只有当附加的ETag与数据库中最新的ETag匹配时，存储才允许更新。
- 当写请求中缺少ETag时，状态存储应以最后写为准（last-write-wins）的方式处理请求。这是为了让高吞吐量写入场景的优化，在这种情况下，数据风险很低或没有负面影响。
- 存储器在向调用者返回状态时，应始终返回ETags。

### 一致性

Dapr允许客户端在get、set和delete操作中附加一致性提示。Dapr支持两个一致性级别：强一致性和最终一致性，定义如下：

#### 最终一致性

Dapr 假定数据存储最终是默认一致性的。状态应该：

- 对于读取请求，状态存储可以从任何一个副本中返回数据。
- 对于写请求，状态存储在确认（ack）更新请求后，应异步复制更新到配置的法定人数。

### 强一致性

当附加了强一致性提示时，状态存储应该：

- 对于读请求，状态存储应该在各个副本中一致地返回最新的数据。
- 对于 write/delete 请求，状态存储应该在完成写请求之前，同步将更新的数据复制到配置的法定人数。

### 例子

以下是一个带有完整操作选项定义的 set 请求示例：

```bash
curl -X POST http://localhost:3500/v1.0/state/starwars \
  -H "Content-Type: application/json" \
  -d '[
        {
          "key": "weapon",
          "value": "DeathStar",
          "etag": "xxxxx",
          "options": {
            "concurrency": "first-write",
            "consistency": "strong",
          }
        }
      ]'
```








