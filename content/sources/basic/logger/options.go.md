---
type: docs
title: "options.go的源码学习"
linkTitle: "options.go"
weight: 113
date: 2021-02-27
description: >
  设置 logger 相关的属性，包括从命令行参数中解析标记
---

Dapr logger package中的 options.go 文件的源码学习，设置logger相关的属性，包括从命令行参数中解析标记。


### 默认属性

```go
const (
	defaultJSONOutput  = false
	defaultOutputLevel = "info"
	undefinedAppID     = ""
)
```

### Options 结构体定义

Options 结构体，就三个字段：

```go
// Options defines the sets of options for Dapr logging
type Options struct {
   // appID is the unique id of Dapr Application
   // 默认为空
   appID string

   // JSONFormatEnabled is the flag to enable JSON formatted log
   // 默认为fasle
   JSONFormatEnabled bool

   // OutputLevel is the level of logging
   // 默认为 info
   OutputLevel string
}
```

### 设值方法

```go
// SetOutputLevel sets the log output level
func (o *Options) SetOutputLevel(outputLevel string) error {
   // 疑问：这里检查和赋值存在不一致：如果 outputLevel 中有大写字母
   // TODO：改进一下
   if toLogLevel(outputLevel) == UndefinedLevel {
      return errors.Errorf("undefined Log Output Level: %s", outputLevel)
   }
   o.OutputLevel = outputLevel
   return nil
}

// SetAppID sets Dapr ID
func (o *Options) SetAppID(id string) {
   o.appID = id
}
```

疑问 ：为什么字段和设置方法不统一？

1.  JSONFormatEnabled 是 public 字段，没有Set方法
2. OutputLevel 是 public 字段，有 Set 方法，Set 方法做了输入值的检测。
   - 问题来了：既然是 public 字段，那么绕开 Set 方法直接赋值岂不是就绕开了输入值检测的逻辑？
3. appID 是 private 字段，有 Set 方法，而 Set 方法什么都没有做，只是简单赋值，那么为什么不直接用 public 字段呢？

检查发现：

- SetOutputLevel 在dapr/dapr 项目中没有任何人调用

### 默认构造

返回每个字段的默认值，没啥特殊：

```go
// DefaultOptions returns default values of Options
func DefaultOptions() Options {
   return Options{
      JSONFormatEnabled: defaultJSONOutput,
      appID:             undefinedAppID,
      OutputLevel:       defaultOutputLevel,
   }
}
```

> 备注：go 不像 java 可以在字段定义时直接赋值一个默认值，有时还真不方便。

### 从命令行标记中读取日志属性

在命令行参数中读取 `log-level` 和 `log-as-json` 两个标记并设置 OutputLevel 和 JSONFormatEnabled：

```go
// AttachCmdFlags attaches log options to command flags
func (o *Options) AttachCmdFlags(
   stringVar func(p *string, name string, value string, usage string),
   boolVar func(p *bool, name string, value bool, usage string)) {
   stringVar(
      &o.OutputLevel,
      "log-level",
      defaultOutputLevel,
      "Options are debug, info, warning, error, or fatal")
   boolVar(
      &o.JSONFormatEnabled,
      "log-as-json",
      defaultJSONOutput,
      "print log as JSON (default false)")
}
```

> 备注：这大概就是 OutputLevel 和 JSONFormatEnabled 两个字段是 public 的原因？

这个方法会在每个二进制文件(runtime(也就是daprd) / injector / operator / placement / sentry) 的初始化代码中调用：

```go
loggerOptions := logger.DefaultOptions()
loggerOptions.AttachCmdFlags(flag.StringVar, flag.BoolVar)
```

> 注意：这个时候 OutputLevel 的值是没有经过检查而直接设值的，绕开了 SetOutputLevel 方法的检查。 

### 将属性应用到所有的logger

```go
// ApplyOptionsToLoggers applys options to all registered loggers
func ApplyOptionsToLoggers(options *Options) error {
   // 所有的 logger 指的是保存在全局 logger map 中所有 logger
   internalLoggers := getLoggers()

   // Apply formatting options first
   for _, v := range internalLoggers {
      v.EnableJSONOutput(options.JSONFormatEnabled)

      if options.appID != undefinedAppID {
         v.SetAppID(options.appID)
      }
   }

   daprLogLevel := toLogLevel(options.OutputLevel)
   if daprLogLevel == UndefinedLevel {
      // 在这里做了 OutputLevel 值的有效性检查
      // 这么报错，只能是从命令行中得到属性然后再设置了
      return errors.Errorf("invalid value for --log-level: %s", options.OutputLevel)
   }

   for _, v := range internalLoggers {
      v.SetOutputLevel(daprLogLevel)
   }
   return nil
}
```

> TODO：OutputLevel 赋值有效性检查的地方现在发现有两个，其中一个还没有被使用。准备PR修订。

查了一下这个方法的确是在每个二进制文件(runtime(也就是daprd) / injector / operator / placement / sentry) 的初始化代码中调用：

```go
loggerOptions := logger.DefaultOptions()
loggerOptions.AttachCmdFlags(flag.StringVar, flag.BoolVar)
......
// Apply options to all loggers
loggerOptions.SetAppID(*appID)
if err := logger.ApplyOptionsToLoggers(&loggerOptions); err != nil {
   return nil, err
}
```

> TODO: ApplyOptionsToLoggers这个方法名最好修改增加“来自命令行的options”语义，否则报错 "invalid value for --log-level“ 就会很奇怪。