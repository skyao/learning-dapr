---
title: "retry.go的源码学习"
linkTitle: "retry.go"
weight: 10
date: 2021-02-27
description: >
  对JSON进行标准化处理
---

Dapr retry package中的 retry.go 文件的源码学习。

### 重试策略

多次重试之间的间隔策略，有两种：PolicyConstant 是固定值，PolicyExponential是指数增长。

```go
// PolicyType 表示后退延迟(back off delay)应该是固定值还是指数增长。
// PolicyType denotes if the back off delay should be constant or exponential.
type PolicyType int

const (
	// PolicyConstant is a backoff policy that always returns the same backoff delay.
    // PolicyConstant是一个总是返回相同退避延迟的退避策略。
	PolicyConstant PolicyType = iota
	// PolicyExponential is a backoff implementation that increases the backoff period
	// for each retry attempt using a randomization function that grows exponentially.
    // PolicyExponential是一个退避实现，它使用一个以指数增长的随机化函数来增加每次重试的退避周期。
	PolicyExponential
)
```

### 重试配置

```go
// Config 封装了退避策略的配置。
type Config struct {
	Policy PolicyType `mapstructure:"policy"`

	// Constant back off
	Duration time.Duration `mapstructure:"duration"`

	// Exponential back off
	InitialInterval     time.Duration `mapstructure:"initialInterval"`
	RandomizationFactor float32       `mapstructure:"randomizationFactor"`
	Multiplier          float32       `mapstructure:"multiplier"`
	MaxInterval         time.Duration `mapstructure:"maxInterval"`
	MaxElapsedTime      time.Duration `mapstructure:"maxElapsedTime"`

	// Additional options
	MaxRetries int64 `mapstructure:"maxRetries"`
}
```

> 注意: 每个字段都标记了 `mapstructure` ，这是为了使用 mapstructure 进行解码。

默认配置为:

```go
func DefaultConfig() Config {
	return Config{
		Policy:              PolicyConstant,		// 默认为固定间隔
		Duration:            5 * time.Second,		// 间隔时间默认是5秒钟
		InitialInterval:     backoff.DefaultInitialInterval,
		RandomizationFactor: backoff.DefaultRandomizationFactor,
		Multiplier:          backoff.DefaultMultiplier,
		MaxInterval:         backoff.DefaultMaxInterval,
		MaxElapsedTime:      backoff.DefaultMaxElapsedTime,
		MaxRetries:          -1,					// 默认一直进行重试
	}
}
```

不带重试的默认配置：

```go
// 这对那些可以自行处理重试的broker来说可能很有用。
func DefaultConfigWithNoRetry() Config {
	c := DefaultConfig()
	c.MaxRetries = 0		// MaxRetries 设置为0

	return c
}
```

### 解码配置

DecodeConfig() 方法将 go 结构体解析为 `Config` : 

```go
func DecodeConfig(c *Config, input interface{}) error {
	// Use the default config if `c` is empty/zero value.
	var emptyConfig Config
	if *c == emptyConfig {		// 如果c是一个初始化之后没有进行赋值的Config结构体，则改用默认配置的Config
		*c = DefaultConfig()
	}

	return config.Decode(input, c)
}
```

DecodeConfigWithPrefix() 方法在将 go 结构体解析为 `Config` 之前，先去除前缀，并进行key和value的正常化: 

```go
func DecodeConfigWithPrefix(c *Config, input interface{}, prefix string) error {
	input, err := config.PrefixedBy(input, prefix)		// 去除前缀，并进行key和value的正常化
	if err != nil {
		return err
	}

	return DecodeConfig(c, input)
}
```

DecodeString()方法解析重试策略：

```go
func (p *PolicyType) DecodeString(value string) error {
	switch strings.ToLower(value) {
	case "constant":
		*p = PolicyConstant
	case "exponential":
		*p = PolicyExponential
	default:
		return errors.Errorf("unexpected back off policy type: %s", value)
	}

	return nil
}
```

### 重试退避时间的生成

NewBackOff() 方法 返回一个 `BackOff` 实例，可直接与`NotifyRecover`或`backoff.RetryNotify`一起使用。该实例不会因为上下文取消而停止。要支持取消（推荐），请使用`NewBackOffWithContext`。 由于底层的回退实现并不总是线程安全的，所以每次使用`RetryNotifyRecover`或`backoff.RetryNotify`时都应该调用`NewBackOff`或`NewBackOffWithContext`。

```go
func (c *Config) NewBackOff() backoff.BackOff {
	var b backoff.BackOff
	switch c.Policy {
	case PolicyConstant:							// 1. 对于固定周期只需要返回配置项中设定的时间间隔，默认5秒钟
		b = backoff.NewConstantBackOff(c.Duration) 
	case PolicyExponential:							// 2. 对于指数周期,通过 backoff 类库来实现，简单透传配置参数
		eb := backoff.NewExponentialBackOff()
		eb.InitialInterval = c.InitialInterval
		eb.RandomizationFactor = float64(c.RandomizationFactor)
		eb.Multiplier = float64(c.Multiplier)
		eb.MaxInterval = c.MaxInterval
		eb.MaxElapsedTime = c.MaxElapsedTime
		b = eb
	}

	if c.MaxRetries >= 0 {
		b = backoff.WithMaxRetries(b, uint64(c.MaxRetries))
	}

	return b
}
```

NewBackOffWithContext() 方法返回一个BackOff实例，以便与`RetryNotifyRecover`或`backoff.RetryNotify`直接使用。如果提供的上下文被取消，则用于取消重试。

由于底层的回退实现并不总是线程安全的，`NewBackOff`或`NewBackOffWithContext`应该在每次使用`RetryNotifyRecover`或`backoff.RetryNotify`时被调用。

```go
func (c *Config) NewBackOffWithContext(ctx context.Context) backoff.BackOff {
	b := c.NewBackOff()

	return backoff.WithContext(b, ctx)
}
```

### 恢复通知

标准  `backoff.RetryNotify`的用法:

```go
func RetryNotify(operation Operation, b BackOff, notify Notify) error {
   return RetryNotifyWithTimer(operation, b, notify, nil)
}

// Operation 是由Retry()或RetryNotify()执行的。
// 如果该操作返回错误，将使用退避策略重试。
type Operation func() error
// Notify是一个出错通知的函数。
// 如果操作失败（有错误），它会收到一个操作错误和回退延迟。
// 注意，如果退避政策要求停止重试。通知函数不会被调用。
type Notify func(error, time.Duration)
```

如果出现问题，需要多次重试才恢复，会存在几个问题：

1. Notify()方法会被调用多次
2. 不好判断是否恢复：理论上"恢复"的概念是先有出错(一次或者连续多次出错)，然后成功（出错之后的第一次不出错）

NotifyRecover() 方法是 `backoff.RetryNotify` 的封装器，它为之前操作失败但后来恢复的情况增加了另一个回调。这个包装器的主要目的是只在操作第一次失败时调用 "notify"，在最后成功时调用 "recovered"。这有助于将日志信息限制在操作者需要被提醒的事件上。

这里的NotifyRecover() 方法包装了 `Operation() `和 `Notify() `函数: 

```go
func NotifyRecover(operation backoff.Operation, b backoff.BackOff, notify backoff.Notify, recovered func()) error {
	var notified bool

	return backoff.RetryNotify(func() error {
		err := operation()

        // notified为true说明之前执行过notify，即出现了一次或者多次连续错误。
        // err为空说明operation不再出错
        // 这才可以成为"恢复"
		if err == nil && notified {	
            notified = false	// 重置 notified ，下一次 operation() 再成功也不会再出发recovered()
            recovered()			// 满足逻辑，可以触发一次 recovered() 方法
		}

		return err
	}, b, func(err error, d time.Duration) {
        if !notified {		// 只在第一次时调用真正的notify()函数，其他情况下忽略
			notify(err, d)
			notified = true
		}
	})
}
```

>  备注：感觉 notified 这个变量的取名不够清晰，它的语义不应该是"是否触发了通知"，而是"是否发生了错误而一直没有恢复"。应该改为类似 errorNotRecoverd 之类的，语义更清晰一些。



