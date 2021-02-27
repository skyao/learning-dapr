---
type: docs
title: "logger.go的源码学习"
linkTitle: "logger.go"
weight: 111
date: 2021-02-27
description: >
  Dapr Logger package中的logger.go文件的源码学习
---

## logger的相关定义

### log type

log类型分为 普通 log 和 request 两种：

```go
const (
	// LogTypeLog is normal log type
	LogTypeLog = "log"
	// LogTypeRequest is Request log type
	LogTypeRequest = "request"
```

### log schema

```go
const (
	// Field names that defines Dapr log schema
	logFieldTimeStamp = "time"
	logFieldLevel     = "level"
	logFieldType      = "type"
	logFieldScope     = "scope"
	logFieldMessage   = "msg"
	logFieldInstance  = "instance"
	logFieldDaprVer   = "ver"
	logFieldAppID     = "app_id"
)
```

### log level

log level 没啥特别，很传统的定义：

```go
const (
	// DebugLevel has verbose message
	DebugLevel LogLevel = "debug"
	// InfoLevel is default log level
	InfoLevel LogLevel = "info"
	// WarnLevel is for logging messages about possible issues
	WarnLevel LogLevel = "warn"
	// ErrorLevel is for logging errors
	ErrorLevel LogLevel = "error"
	// FatalLevel is for logging fatal messages. The system shuts down after logging the message.
	FatalLevel LogLevel = "fatal"

	// UndefinedLevel is for undefined log level
	UndefinedLevel LogLevel = "undefined"
)
```

注意： FatalLevel 有特别的意义，”The system shuts down after logging the message“.  所以这个不能随便用。

toLogLevel 将字符串转为 LogLevel，大小写不敏感：

```go
// toLogLevel converts to LogLevel
func toLogLevel(level string) LogLevel {
	switch strings.ToLower(level) {
	case "debug":
		return DebugLevel
	case "info":
		return InfoLevel
	case "warn":
		return WarnLevel
	case "error":
		return ErrorLevel
	case "fatal":
		return FatalLevel
	}

	// unsupported log level by Dapr
	return UndefinedLevel
}
```

### Logger 接口定义

```go

// Logger includes the logging api sets
type Logger interface {
	// EnableJSONOutput enables JSON formatted output log
	EnableJSONOutput(enabled bool)

	// SetAppID sets dapr_id field in log. Default value is empty string
	SetAppID(id string)
	// SetOutputLevel sets log output level
	SetOutputLevel(outputLevel LogLevel)

	// WithLogType specify the log_type field in log. Default value is LogTypeLog
	WithLogType(logType string) Logger

	// Info logs a message at level Info.
	Info(args ...interface{})
	// Infof logs a message at level Info.
	Infof(format string, args ...interface{})
	// Debug logs a message at level Debug.
	Debug(args ...interface{})
	// Debugf logs a message at level Debug.
	Debugf(format string, args ...interface{})
	// Warn logs a message at level Warn.
	Warn(args ...interface{})
	// Warnf logs a message at level Warn.
	Warnf(format string, args ...interface{})
	// Error logs a message at level Error.
	Error(args ...interface{})
	// Errorf logs a message at level Error.
	Errorf(format string, args ...interface{})
	// Fatal logs a message at level Fatal then the process will exit with status set to 1.
	Fatal(args ...interface{})
	// Fatalf logs a message at level Fatal then the process will exit with status set to 1.
	Fatalf(format string, args ...interface{})
}
```

## logger的创建和获取

### 全集 log 列表

```go
// globalLoggers is the collection of Dapr Logger that is shared globally.
// TODO: User will disable or enable logger on demand.
var globalLoggers = map[string]Logger{}
var globalLoggersLock = sync.RWMutex{}
```

### 创建新logger或获取已经保存的logger

logger创建之后会保存在 global loggers 中，这意味着每个 name 的logger只会创建一个实例。

```go
// NewLogger creates new Logger instance.
func NewLogger(name string) Logger {
	globalLoggersLock.Lock()
	defer globalLoggersLock.Unlock()

	logger, ok := globalLoggers[name]
	if !ok {
		logger = newDaprLogger(name)
		globalLoggers[name] = logger
	}

	return logger
}
```

newDaprLogger() 方法的细节见

### 获取所有已经创建的logger列表

```go
func getLoggers() map[string]Logger {
	globalLoggersLock.RLock()
	defer globalLoggersLock.RUnlock()

	l := map[string]Logger{}
	for k, v := range globalLoggers {
		l[k] = v
	}

	return l
}
```
