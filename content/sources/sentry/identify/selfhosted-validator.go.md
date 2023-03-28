---
title: "selfhosted下的validator.go"
linkTitle: "selfhosted"
weight: 50
date: 2022-02-22
description: >
  selfhosted下的validator实现
---



selfhosted 下实际没有做验证：

```go
func NewValidator() identity.Validator {
	return &validator{}
}

type validator struct{}

func (v *validator) Validate(id, token, namespace string) error {
	// no validation for self hosted.
	return nil
}
```

只是保留了一套代码框架，以满足 Validator 接口的要求。

这意味着在 selfhosted 下是不会进行身份验证的。
