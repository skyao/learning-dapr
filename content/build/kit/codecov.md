---
title: "codedov文件"
linkTitle: "codedov"
weight: 20
date: 2022-01-21
description: >
  codedov的配置文件
---

## codecov 背景知识

https://docs.codecov.com/docs/codecov-yaml

Codecov Yaml文件是唯一的配置点，为开发者提供了一个透明的、版本控制的文件来调整所有的Codecov设置。

参考手册：

https://docs.codecov.com/docs/codecovyml-reference

验证工具：

https://api.codecov.io/validate

## kit仓库配置

https://github.com/dapr/kit/blob/main/.codecov.yaml

kit仓库的 .codecov.yaml 文件极其简单，只配置了 `coverage.status` 的少数属性：

```yaml
coverage:
  # Commit status https://docs.codecov.io/docs/commit-status are used
  # to block PR based on coverage threshold.
  status:
    project:
      default:
        target: auto
        threshold: 0%
    patch:
      default:
        informational: true
```

参考 https://docs.codecov.com/docs/commit-status 中的说明, status 检查的功能是:

> Useful for blocking Pull Requests that don't meet a particular coverage threshold.
>
> 有助于阻止不符合特定覆盖率阈值的Pull Requests。

### target

取值可以是 `auto` 或者 `number` 。

选择一个最小的覆盖率，提交必须满足该覆盖率才能被认为是成功的。

- `auto` 将使用基础提交（PR请求基础或父提交）的覆盖率来进行比较。
- `number` 可以指定一个精确的覆盖率目标，如75%或100%（接受字符串、int或float）。

### threshold

取值为  `number` 。

允许覆盖率下降 `X%`，并发布成功状态。

### informational

在信息模式下使用Codecov。默认为false。如果指定为 "true"，那么无论覆盖率是多少或其他设置是什么，结果都会通过。如果想在PR中向其他开发者公开codecov信息，而不一定要在这些信息上进行门禁，那么信息模式是非常好的选择。

## 本地执行

本地执行go test 命令，加上相关的参数之后就可以得到覆盖率信息：

```bash
$ go test ./... -coverprofile=coverage.txt -covermode=atomic
ok      github.com/dapr/kit/config      0.008s  coverage: 95.5% of statements
ok      github.com/dapr/kit/logger      0.008s  coverage: 87.0% of statements
ok      github.com/dapr/kit/retry       0.006s  coverage: 97.6% of statements
```

可以用工具查看更详细的输出：

```bash
go tool cover -html=coverage.txt
```

如果是使用IDE如 goland，可以参考：

https://www.jetbrains.com/help/go/code-coverage.html

最常见的方式是在编写单元测试案例时，右键 "More Run/Debug" -> "Run xxx.go.test with Coverage"，执行结束之后会自动打开 Coverage 视图。

## CI执行

可以通过下面的地址查看：

https://app.codecov.io/gh/dapr/kit

CI执行 Coverage 检查的过程，请参见下一章 workflow 的详细说明。
