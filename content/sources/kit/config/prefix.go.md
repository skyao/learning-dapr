---
title: "prefix.go的源码学习"
linkTitle: "prefix.go"
weight: 30
date: 2021-02-27
description: >
  去除key的前缀
---

Dapr config package中的 prefix.go 文件的源码学习。

### 代码实现

```go
func PrefixedBy(input interface{}, prefix string) (interface{}, error) {
	normalized, err := Normalize(input)
	if err != nil {
        // 唯一可能来自normalize的错误是: 输入是map[interface{}]interface{}，而某个key不是字符串
		return input, err
	}
	input = normalized

	if inputMap, ok := input.(map[string]interface{}); ok {
		converted := make(map[string]interface{}, len(inputMap))
		for k, v := range inputMap {
			if strings.HasPrefix(k, prefix) {
				key := uncapitalize(strings.TrimPrefix(k, prefix)) // 去掉key的前缀
				converted[key] = v
			}
		}

		return converted, nil
	} else if inputMap, ok := input.(map[string]string); ok {
		converted := make(map[string]string, len(inputMap))
		for k, v := range inputMap {
			if strings.HasPrefix(k, prefix) {
				key := uncapitalize(strings.TrimPrefix(k, prefix)) // 去掉key的前缀
				converted[key] = v
			}
		}

		return converted, nil
	}

	return input, nil
}
```

uncapitalize()方法将字符串转为小写：

```go
func uncapitalize(str string) string {
	if len(str) == 0 {
		return str
	}

	vv := []rune(str) // Introduced later
	vv[0] = unicode.ToLower(vv[0])

	return string(vv)
}
```

### 使用场景

被 retry.go 的 DecodeConfigWithPrefix() 方法调用

```go
func DecodeConfigWithPrefix(c *Config, input interface{}, prefix string) error {
	input, err := config.PrefixedBy(input, prefix)
	if err != nil {
		return err
	}

	return DecodeConfig(c, input)
}
```

