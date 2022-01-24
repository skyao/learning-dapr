---
title: "codedov文件"
linkTitle: "codedov"
weight: 20
date: 2022-01-24
description: >
  codedov的配置文件
---

## components-contrib仓库配置

https://github.com/dapr/components-contrib/blob/master/.codecov.yaml

kit仓库的 .codecov.yaml 文件极其简单，只配置了 `coverage.status` 的少数属性：

```yaml
coverage:
  # Commit status https://docs.codecov.io/docs/commit-status are used
  # to block PR based on coverage threshold.
  status:
    project:
      default:
        informational: true
    patch:
      # Disable the coverage threshold of the patch, so that PRs are
      # only failing because of overall project coverage threshold.
      # See https://docs.codecov.io/docs/commit-status#disabling-a-status.
      default: false
comment:
  # Delete old comment and post new one for new coverage information.
  behavior: new
```

和 kit 仓库的很不一样。

TODO： 后面来看为啥差别这么大。

