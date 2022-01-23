---
title: "normalize.go的源码学习"
linkTitle: "normalize.go"
weight: 20
date: 2021-02-27
description: >
  对JSON进行标准化处理
---

Dapr config package中的 normalize.go 文件的源码学习。

将 `map[interface{}]interface{}` 转换为 `map[string]interface{}`，以便对JSON进行标准化处理，并在组件初始化时使用。

代码实现:

```go
func Normalize(i interface{}) (interface{}, error) {
	var err error
	switch x := i.(type) {				// 只标准化三种类型，其他类型直接返回
	case map[interface{}]interface{}:	// 1. 对于map[interface{}]interface{}，key和value都要做正常化
		m2 := map[string]interface{}{}
		for k, v := range x {
			if strKey, ok := k.(string); ok {
                // 将key的类型改成string，value继续做正常化
				if m2[strKey], err = Normalize(v); err != nil {
					return nil, err
				}
			} else {
                // 要求key一定是string，否则报错
				return nil, fmt.Errorf("error parsing config field: %v", k)
			}
		}

		return m2, nil
	case map[string]interface{}:		// 2. 对于map[string{}]interface{}，只需要对value做正常化
		m2 := map[string]interface{}{}
		for k, v := range x {
			if m2[k], err = Normalize(v); err != nil {
				return nil, err
			}
		}

		return m2, nil
	case []interface{}:					// 3. 对于[]interface{}这样的数组，每个数组元素都做正常化
		for i, v := range x {
			if x[i], err = Normalize(v); err != nil {
				return nil, err
			}
		}
	}

	return i, nil
}
```

