---
type: docs
title: "dapr_logger.go的源码学习"
linkTitle: "dapr_logger.go"
weight: 212
date: 2021-02-27
description: >
  daprLogger 是实际的日志实现
---

Dapr logger package中的dapr_logger.go文件的源码分析，daprLogger 是实际的日志实现。

### daprLogger 结构体定义

daprLogger 结构体，底层实现是 logrus ： 

```go
// daprLogger is the implemention for logrus
type daprLogger struct {
	// name is the name of logger that is published to log as a scope
	name string
	// loger is the instance of logrus logger
	logger *logrus.Entry
}
```

### 创建Dapr logger

创建Dapr logger的逻辑：

```go
func newDaprLogger(name string) *daprLogger {
   // 底层是 logrus
	newLogger := logrus.New()
   // 输出到 stdout
	newLogger.SetOutput(os.Stdout)

	dl := &daprLogger{
		name: name,
		logger: newLogger.WithFields(logrus.Fields{
			logFieldScope: name,
         // 默认是普通log类型
			logFieldType:  LogTypeLog,
		}),
	}

   // 设置是否启用json输出，defaultJSONOutput默认是false
	dl.EnableJSONOutput(defaultJSONOutput)

	return dl
}
```

### 启用json输出

函数名有点小问题，实际是初始化logger，是否enables JSON只是部分逻辑：

```go
// EnableJSONOutput enables JSON formatted output log
func (l *daprLogger) EnableJSONOutput(enabled bool) {
	var formatter logrus.Formatter

	fieldMap := logrus.FieldMap{
		// If time field name is conflicted, logrus adds "fields." prefix.
		// So rename to unused field @time to avoid the confliction.
		logrus.FieldKeyTime:  logFieldTimeStamp,
		logrus.FieldKeyLevel: logFieldLevel,
		logrus.FieldKeyMsg:   logFieldMessage,
	}

	hostname, _ := os.Hostname()
	l.logger.Data = logrus.Fields{
		logFieldScope:    l.logger.Data[logFieldScope],
		logFieldType:     LogTypeLog,
		logFieldInstance: hostname,
		logFieldDaprVer:  version.Version(),
	}

	if enabled {
		formatter = &logrus.JSONFormatter{
			TimestampFormat: time.RFC3339Nano,
			FieldMap:        fieldMap,
		}
	} else {
		formatter = &logrus.TextFormatter{
			TimestampFormat: time.RFC3339Nano,
			FieldMap:        fieldMap,
		}
	}

	l.logger.Logger.SetFormatter(formatter)
}
```

TODO：考虑重构

## logger的设置

### 设置appid

设置日志的 app_id 字段，默认为空。

```go
// SetAppID sets app_id field in log. Default value is empty string
func (l *daprLogger) SetAppID(id string) {
	l.logger = l.logger.WithField(logFieldAppID, id)
}
```

### 设置日志级别

```go
// SetOutputLevel sets log output level
func (l *daprLogger) SetOutputLevel(outputLevel LogLevel) {
   l.logger.Logger.SetLevel(toLogrusLevel(outputLevel))
}

func toLogrusLevel(lvl LogLevel) logrus.Level {
	// ignore error because it will never happens
	l, _ := logrus.ParseLevel(string(lvl))
	return l
}
```

这个是在原有的 daprLogger 实例上进行设置，没啥特殊。

### 设置日志类型

默认是普通 log 类型，如果要设置log类型：

```go
// WithLogType specify the log_type field in log. Default value is LogTypeLog
func (l *daprLogger) WithLogType(logType string) Logger {
   // 这里重新构造了一个新的 daprLogger 结构体，然后返回？？
   return &daprLogger{
      name:   l.name,
      logger: l.logger.WithField(logFieldType, logType),
   }
}
```

疑问和TODO：

1. 为什么不是直接设置 l.logger，而是构造一个新的结构体，然后返回还是 Logger ？
2. 会不会有隐患？前面logger创建之后是存放在global logger map中的，key是简单的 name 而不是 name + logtype，这岂不是无法保存一个 name 两个类型的两个 logger 对象？

## logger的实现

所有的写log的方法都简单代理给了 l.logger （*logrus.Entry）：

```go
// Info logs a message at level Info.
func (l *daprLogger) Info(args ...interface{}) {
	l.logger.Log(logrus.InfoLevel, args...)
}

// Infof logs a message at level Info.
func (l *daprLogger) Infof(format string, args ...interface{}) {
	l.logger.Logf(logrus.InfoLevel, format, args...)
}

// Debug logs a message at level Debug.
func (l *daprLogger) Debug(args ...interface{}) {
	l.logger.Log(logrus.DebugLevel, args...)
}

// Debugf logs a message at level Debug.
func (l *daprLogger) Debugf(format string, args ...interface{}) {
	l.logger.Logf(logrus.DebugLevel, format, args...)
}

// Warn logs a message at level Warn.
func (l *daprLogger) Warn(args ...interface{}) {
	l.logger.Log(logrus.WarnLevel, args...)
}

// Warnf logs a message at level Warn.
func (l *daprLogger) Warnf(format string, args ...interface{}) {
	l.logger.Logf(logrus.WarnLevel, format, args...)
}

// Error logs a message at level Error.
func (l *daprLogger) Error(args ...interface{}) {
	l.logger.Log(logrus.ErrorLevel, args...)
}

// Errorf logs a message at level Error.
func (l *daprLogger) Errorf(format string, args ...interface{}) {
	l.logger.Logf(logrus.ErrorLevel, format, args...)
}

// Fatal logs a message at level Fatal then the process will exit with status set to 1.
func (l *daprLogger) Fatal(args ...interface{}) {
	l.logger.Fatal(args...)
}

// Fatalf logs a message at level Fatal then the process will exit with status set to 1.
func (l *daprLogger) Fatalf(format string, args ...interface{}) {
	l.logger.Fatalf(format, args...)
}
```

