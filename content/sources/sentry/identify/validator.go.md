---
title: "validator.go"
linkTitle: "validator"
weight: 20
date: 2022-02-22
description: >
  validator 接口定义
---



### 接口定义

Validator 通过使用 ID 和 token 来验证证书请求的身份

```go
type Validator interface {
	Validate(id, token, namespace string) error
}
```



