---
title: "decode.go的源码学习"
linkTitle: "decode.go"
weight: 10
date: 2021-02-27
description: >
  从config中解析出配置信息。
---

Dapr config package中的 decode.go 文件的源码学习。

## Decoder的相关定义

### StringDecoder 

```go
// StringDecoder被用作自定义类型（或别名类型）来覆盖 `decodeString` DecodeHook中的基本解码功能的一种方式。 
// `encoding.TextMashaller`没有被使用，是因为它与许多Go类型相匹配，并且会有潜在的意外结果。
// 指定一个自定义的解码func应该是非常有意的。
type StringDecoder interface {
	DecodeString(value string) error
}
```

### Decode()方法

```go
// Decode()将通用map值从 `input` 解码到 `output`，同时提供有用的错误信息。
// `output`必须是一个指向Go结构体的指针，该结构体包含应被解码的字段的 `mapstructure` 结构体标签。
// 这个函数在解码被解析为 `map[string]interface{}` 的配置文件或被解析为`map[string]string` 的组件元数据的值时很有用。
// 
// 大部分繁重的工作都由 mapstructure 库处理。自定义的解码器被用来处理将字符串值解码为支持的原生类型。
func Decode(input interface{}, output interface{}) error {
	// 构建mapstructure的decoder
	decoder, err := mapstructure.NewDecoder(&mapstructure.DecoderConfig{ // nolint:exhaustivestruct
		Result:     output,
		DecodeHook: decodeString,	// 这里植入我们的hook
	})
	if err != nil {
		return err
	}
	// 委托给mapstructure的decoder进行解码
	return decoder.Decode(input)
}
```

DecodeHookFunc 的定义：

```
type DecodeHookFunc interface{}
```

DecodeHookFunc() 要求必须是下面的三个方法之一：

 ```go
 // DecodeHookFuncType is a DecodeHookFunc which has complete information about
// the source and target types.
type DecodeHookFuncType func(reflect.Type, reflect.Type, interface{}) (interface{}, error)

// DecodeHookFuncKind is a DecodeHookFunc which knows only the Kinds of the
// source and target types.
type DecodeHookFuncKind func(reflect.Kind, reflect.Kind, interface{}) (interface{}, error)

// DecodeHookFuncRaw is a DecodeHookFunc which has complete access to both the source and target
// values.
type DecodeHookFuncValue func(from reflect.Value, to reflect.Value) (interface{}, error)
 ```

config实现中采用的是第一种： 有 source 和 target 类型的完整信息。

### decodeString()方法

decodeString()方法的实现：

```go
func decodeString(
	f reflect.Type,
	t reflect.Type,
	data interface{}) (interface{}, error) {
	if t.Kind() == reflect.String && f.Kind() != reflect.String {
		return fmt.Sprintf("%v", data), nil
	}
	if f.Kind() == reflect.Ptr {
		f = f.Elem()
		data = reflect.ValueOf(data).Elem().Interface()
	}
	if f.Kind() != reflect.String {
		return data, nil
	}

	dataString, ok := data.(string)
	if !ok {
		return nil, errors.Errorf("expected string: got %s", reflect.TypeOf(data))
	}

	var result interface{}
	var decoder StringDecoder

	if t.Implements(typeStringDecoder) {
		result = reflect.New(t.Elem()).Interface()
		decoder = result.(StringDecoder)
	} else if reflect.PtrTo(t).Implements(typeStringDecoder) {
		result = reflect.New(t).Interface()
		decoder = result.(StringDecoder)
	}

	if decoder != nil {
		if err := decoder.DecodeString(dataString); err != nil {
			if t.Kind() == reflect.Ptr {
				t = t.Elem()
			}

			return nil, errors.Errorf("invalid %s %q: %v", t.Name(), dataString, err)
		}

		return result, nil
	}

	switch t {
	case typeDuration:
		// Check for simple integer values and treat them
		// as milliseconds
		if val, err := strconv.Atoi(dataString); err == nil {
			return time.Duration(val) * time.Millisecond, nil
		}

		// Convert it by parsing
		d, err := time.ParseDuration(dataString)

		return d, invalidError(err, "duration", dataString)
	case typeTime:
		// Convert it by parsing
		t, err := time.Parse(time.RFC3339Nano, dataString)
		if err == nil {
			return t, nil
		}
		t, err = time.Parse(time.RFC3339, dataString)

		return t, invalidError(err, "time", dataString)
	}

	switch t.Kind() { // nolint: exhaustive
	case reflect.Uint:
		val, err := strconv.ParseUint(dataString, 10, 64)

		return uint(val), invalidError(err, "uint", dataString)
	case reflect.Uint64:
		val, err := strconv.ParseUint(dataString, 10, 64)

		return val, invalidError(err, "uint64", dataString)
	case reflect.Uint32:
		val, err := strconv.ParseUint(dataString, 10, 32)

		return uint32(val), invalidError(err, "uint32", dataString)
	case reflect.Uint16:
		val, err := strconv.ParseUint(dataString, 10, 16)

		return uint16(val), invalidError(err, "uint16", dataString)
	case reflect.Uint8:
		val, err := strconv.ParseUint(dataString, 10, 8)

		return uint8(val), invalidError(err, "uint8", dataString)

	case reflect.Int:
		val, err := strconv.ParseInt(dataString, 10, 64)

		return int(val), invalidError(err, "int", dataString)
	case reflect.Int64:
		val, err := strconv.ParseInt(dataString, 10, 64)

		return val, invalidError(err, "int64", dataString)
	case reflect.Int32:
		val, err := strconv.ParseInt(dataString, 10, 32)

		return int32(val), invalidError(err, "int32", dataString)
	case reflect.Int16:
		val, err := strconv.ParseInt(dataString, 10, 16)

		return int16(val), invalidError(err, "int16", dataString)
	case reflect.Int8:
		val, err := strconv.ParseInt(dataString, 10, 8)

		return int8(val), invalidError(err, "int8", dataString)

	case reflect.Float32:
		val, err := strconv.ParseFloat(dataString, 32)

		return float32(val), invalidError(err, "float32", dataString)
	case reflect.Float64:
		val, err := strconv.ParseFloat(dataString, 64)

		return val, invalidError(err, "float64", dataString)

	case reflect.Bool:
		val, err := strconv.ParseBool(dataString)

		return val, invalidError(err, "bool", dataString)

	default:
		return data, nil
	}
}
```