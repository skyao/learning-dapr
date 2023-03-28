---
title: "kubernetes下的validator.go"
linkTitle: "kubernetes"
weight: 40
date: 2022-02-22
description: >
  kubernetes下的validator实现
---

## 准备工作

### 结构体定义

validator 结构体定义：

```go
type validator struct {
	client    k8s.Interface
	auth      kauth.AuthenticationV1Interface
	audiences []string
}
```



### 创建validator的方法

NewValidator() 方法创建新的 validator 结构体：

```go
func NewValidator(client k8s.Interface, audiences []string) identity.Validator {
	return &validator{
		client:    client,
		auth:      client.AuthenticationV1(),
		audiences: audiences,
	}
}
```

## 实现

Validate() 实现通过使用 ID 和 token 来验证证书请求的身份：

```go
func (v *validator) Validate(id, token, namespace string) error {
  // id 和 token 不能为空
	if id == "" {
		return fmt.Errorf("%s: id field in request must not be empty", errPrefix)
	}
	if token == "" {
		return fmt.Errorf("%s: token field in request must not be empty", errPrefix)
	}

	// TODO: Remove in Dapr 1.12 to enforce setting an explicit audience
	var canTryWithNilAudience, showDefaultTokenAudienceWarning bool

	audiences := v.audiences
  
	if len(audiences) == 0 {
    // 处理用户没有显式设置 audience 的特殊情况
    // 此时采用默认是 sentryConsts.ServiceAccountTokenAudience "dapr.io/sentry"
		audiences = []string{sentryConsts.ServiceAccountTokenAudience}

		// TODO: Remove in Dapr 1.12 to enforce setting an explicit audience
		// Because the user did not specify an explicit audience and is instead relying on the default, if the authentication fails we can retry with nil audience
    // 并记录下来这是特殊情况，如果认证失败则应该尝试 audience 为 nil 的情况
		canTryWithNilAudience = true
	}
	tokenReview := &kauthapi.TokenReview{
		Spec: kauthapi.TokenReviewSpec{
			Token:     token,
			Audiences: audiences,
		},
	}

tr: // TODO: Remove in Dapr 1.12 to enforce setting an explicit audience

	prts, err := v.executeTokenReview(tokenReview)
	if err != nil {
		// TODO: Remove in Dapr 1.12 to enforce setting an explicit audience
		if canTryWithNilAudience {
			// Retry with a nil audience, which means the default audience for the K8s API server
			tokenReview.Spec.Audiences = nil
			showDefaultTokenAudienceWarning = true
			canTryWithNilAudience = false
			goto tr
		}

		return err
	}

	// TODO: Remove in Dapr 1.12 to enforce setting an explicit audience
	if showDefaultTokenAudienceWarning {
		log.Warn("WARNING: Sentry accepted a token with the audience for the Kubernetes API server. This is deprecated and only supported to ensure a smooth upgrade from Dapr pre-1.10.")
	}

	if len(prts) != 4 || prts[0] != "system" {
		return fmt.Errorf("%s: provided token is not a properly structured service account token", errPrefix)
	}

	podSa := prts[3]
	podNs := prts[2]

  // 检验 namespace
	if namespace != "" {
		if podNs != namespace {
			return fmt.Errorf("%s: namespace mismatch. received namespace: %s", errPrefix, namespace)
		}
	}

  // 检验 id
	if id != podNs+":"+podSa {
		return fmt.Errorf("%s: token/id mismatch. received id: %s", errPrefix, id)
	}
	return nil
}
```

executeTokenReview() 方法执行 tokenReview，如果 token 无效或者失败则返回错误：

```go
func (v *validator) executeTokenReview(tokenReview *kauthapi.TokenReview) ([]string, error) {
	review, err := v.auth.TokenReviews().Create(context.TODO(), tokenReview, v1.CreateOptions{})
	if err != nil {
		return nil, fmt.Errorf("%s: token review failed: %w", errPrefix, err)
	}
	if review.Status.Error != "" {
		return nil, fmt.Errorf("%s: invalid token: %s", errPrefix, review.Status.Error)
	}
	if !review.Status.Authenticated {
		return nil, fmt.Errorf("%s: authentication failed", errPrefix)
	}
	return strings.Split(review.Status.User.Username, ":"), nil
}
```

