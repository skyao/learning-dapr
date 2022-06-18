---
title: "性能测试案例 service invoke http 的实现"
linkTitle: "service invoke http"
weight: 100
date: 2021-05-09
description: >
  性能测试案例 service invoke http 的实现
---

## 运行性能测试

打开 dapr/dapr 仓库下的 `.github/workflows/dapr-perf.yml` 文件，找到 service_invocation_http 的性能测试输入条件：

```yaml
      - name: Run Perf test service_invocation_http
        if: env.TEST_PREFIX != ''
        env:
          DAPR_PERF_QPS: 1000
          DAPR_PERF_CONNECTIONS: 16
          DAPR_TEST_DURATION: 1m
          DAPR_PAYLOAD_SIZE: 1024
        run: make test-perf-service_invocation_http
```



```bash
# service_invocation_http
export DAPR_PERF_QPS=1000
export DAPR_PERF_CONNECTIONS=16
export DAPR_TEST_DURATION=1m
export DAPR_PAYLOAD_SIZE=1024
unset DAPR_PAYLOAD
make test-perf-service_invocation_http
```





## 主流程详细分析

性能测试的主流程为：

``` go
func TestMain(m *testing.M) {
  // 步骤1: 准备两个app：作为服务器端的 testapp 和作为客户端的 tester
	testApps := []kube.AppDescription{
		{
			AppName:           "testapp",
		},
		{
			AppName:           "tester",
	}

	tr = runner.NewTestRunner("serviceinvocationhttp", testApps, nil, nil)
	os.Exit(tr.Start(m))
}
  
func TestServiceInvocationHTTPPerformance(t *testing.T) {
  // 步骤4: 执行测试案例 
......
}
 

// Start is the entry point of Dapr test runner.
func (tr *TestRunner) Start(m runnable) int {
	// 步骤2: 启动测试平台
	err := tr.Platform.setup()

  // 可选步骤2.5: 安装组件，这个测试案例中没有
	if tr.components != nil && len(tr.components) > 0 {
		log.Println("Installing components...")
		if err := tr.Platform.addComponents(tr.components); err != nil {
			fmt.Fprintf(os.Stderr, "Failed Platform.addComponents(), %s", err.Error())
			return runnerFailExitCode
		}
	}

  // 可选步骤2.75: 安装初始化应用，这个测试案例中没有
	if tr.initApps != nil && len(tr.initApps) > 0 {
		log.Println("Installing init apps...")
		if err := tr.Platform.addApps(tr.initApps); err != nil {
			fmt.Fprintf(os.Stderr, "Failed Platform.addInitApps(), %s", err.Error())
			return runnerFailExitCode
		}
	}

	// 步骤3: 安装测试应用，这个测试案例中是前面步骤1中准备的作为服务器端的 testapp 和作为客户端的 tester
	if tr.testApps != nil && len(tr.testApps) > 0 {
		log.Println("Installing test apps...")
		if err := tr.Platform.addApps(tr.testApps); err != nil {
			fmt.Fprintf(os.Stderr, "Failed Platform.addApps(), %s", err.Error())
			return runnerFailExitCode
		}
	}

	// 步骤4: 执行测试案例 
	return m.Run()
}
  
func (tr *TestRunner) tearDown() {
  // 步骤5: 执行完成后的 tearDown
  // 具体为删除前面步骤3中安装的测试应用（作为服务器端的 testapp 和作为客户端的 tester）
  // 如果用 ctrl + c 等方式强行中断 testcase 的执行，就会导致 teardown 没有执行，
  // testapp/tester 两个应用就不会从k8s中删除
	tr.Platform.tearDown()
}
```

### 测试应用准备

对应到上面主流程中的 "步骤1: 准备两个app", 这里需要准备作为服务器端的 testapp 和作为客户端的 tester:

```go
testApps := []kube.AppDescription{
		{
			AppName:           "testapp",				// 作为服务器端的 testapp 
			DaprEnabled:       true,
			ImageName:         "perf-service_invocation_http",
			IngressEnabled:    true,
			......
		},
		{
			AppName:           "tester",				// 作为客户端的 tester
			DaprEnabled:       true,
			ImageName:         "perf-tester",
			IngressEnabled:    true,
			AppPort:           3001,
			......
		},
	}
```

特别注意：`IngressEnabled:    true` 的设置。

### 测试应用安装的流程

对应到上面主流程中的 "步骤3: 安装测试应用"。



对应代码在 `tests/runner/kube_testplatform.go` 中的 addApps() 方法：

```go
// addApps adds test apps to disposable App Resource queues.
func (c *KubeTestPlatform) addApps(apps []kube.AppDescription) error {
 
  	for _, app := range apps {
      log.Printf("Adding app %v", app)
			c.AppResources.Add(kube.NewAppManager(c.KubeClient, getNamespaceOrDefault(app.Namespace), app))
    }
  
  	// installApps installs the apps in AppResource queue sequentially
	log.Printf("Installing apps ...")
	if err := c.AppResources.setup(); err != nil {
		return err
	}
	log.Printf("Apps are installed.")
}
```

添加 app 这里对应的日志为：

```bash
2022/03/27 11:48:39 Adding app {testapp 0  map[] true perf-service_invocation_http:dev-linux-amd64  docker.io/skyao 1 true true   4.0 0.1 800Mi 2500Mi 4.0 0.1 512Mi 250Mi <nil> false}
2022/03/27 11:48:39 Adding app {tester 3001  map[] true perf-tester:dev-linux-amd64  docker.io/skyao 1 true true   4.0 0.1 800Mi 2500Mi 4.0 0.1 512Mi 250Mi <nil> false}
```

遇到问题时建议特别小心的检查这行日志，确认各个参数（如 image 的 name, tag，registry）等是否OK。

### 安装测试应用

详细看安装测试应用的代码和日志：

```go
log.Printf("Deploying app %v ...", m.app.AppName)

		// Deploy app and wait until deployment is done
		if _, err := m.Deploy(); err != nil {
			return err
		}

		// Wait until app is deployed completely
		if _, err := m.WaitUntilDeploymentState(m.IsDeploymentDone); err != nil {
			return err
		}

		if m.logPrefix != "" {
			if err := m.StreamContainerLogs(); err != nil {
				log.Printf("Failed to retrieve container logs for %s. Error was: %s", m.app.AppName, err)
			}
		}
	
	log.Printf("App %v has been deployed.", m.app.AppName)
```

对应日志为：

```bash
2022/03/27 11:48:41 Deploying app testapp ...
2022/03/27 11:48:48 App testapp has been deployed.

2022/03/27 11:48:50 Deploying app tester ...
2022/03/27 11:48:57 App tester has been deployed.
```

从日志上看，在镜像本地已经有缓存的情况下，启动时间也就大概7秒左右。

```go
	// PollInterval is how frequently e2e tests will poll for updates.
	PollInterval = 1 * time.Second
	// PollTimeout is how long e2e tests will wait for resource updates when polling.
	PollTimeout = 10 * time.Minute

// WaitUntilDeploymentState waits until isState returns true.
func (m *AppManager) WaitUntilDeploymentState(isState func(*appsv1.Deployment, error) bool) (*appsv1.Deployment, error) {
	deploymentsClient := m.client.Deployments(m.namespace)

	var lastDeployment *appsv1.Deployment

	waitErr := wait.PollImmediate(PollInterval, PollTimeout, func() (bool, error) {
		var err error
		lastDeployment, err = deploymentsClient.Get(context.TODO(), m.app.AppName, metav1.GetOptions{})
		done := isState(lastDeployment, err)
		if !done && err != nil {
			return true, err
		}
		return done, nil
	})

	if waitErr != nil {
		// get deployment's Pods detail status info
		......
	}

	return lastDeployment, nil
}
```

从代码实现看，等待应用部署的时间长达10分钟，所以正常情况下还是能等到应用启动完成的，除非应用的部署出问题了，比如镜像信息不对导致无法下载镜像。

### 检查 sidecar

检查 sidecar 是否启动成功的代码：

```go
	// maxSideCarDetectionRetries is the maximum number of retries to detect Dapr sidecar.
	maxSideCarDetectionRetries = 3

log.Printf("Validating sidecar for app %v ....", m.app.AppName)
		for i := 0; i <= maxSideCarDetectionRetries; i++ {
			// Validate daprd side car is injected
			if err := m.ValidateSidecar(); err != nil {
				if i == maxSideCarDetectionRetries {
					return err
				}

				log.Printf("Did not find sidecar for app %v error %s, retrying ....", m.app.AppName, err)
				time.Sleep(10 * time.Second)
				continue
			}

			break
		}
		log.Printf("Sidecar for app %v has been validated.", m.app.AppName)
```

对应的日志为：

```bash
2022/03/27 11:48:48 Validating sidecar for app testapp ....
2022/03/27 11:48:48 Streaming Kubernetes logs to ./container_logs/testapp-85d8d9db89-llmrr.daprd.log
2022/03/27 11:48:48 Streaming Kubernetes logs to ./container_logs/testapp-85d8d9db89-llmrr.testapp.log
2022/03/27 11:48:49 Sidecar for app testapp has been validated.

2022/03/27 11:48:57 Validating sidecar for app tester ....
2022/03/27 11:48:57 Streaming Kubernetes logs to ./container_logs/tester-7944b6bb68-wzfj2.tester.log
2022/03/27 11:48:57 Streaming Kubernetes logs to ./container_logs/tester-7944b6bb68-wzfj2.daprd.log
2022/03/27 11:48:58 Sidecar for app tester has been validated.
```

默认重试3次，每次间隔时间为 10 秒，所以执行4次 （1次 + 3次重试）总共40 秒之后，如果 sidecar 还没能启动起来，就会报错。

由于 sidecar 几乎是和应用同时启动，所以在应用启动完成后在检查 sidecar 通常会很快完成，由于应用自身启动时间就高达7秒。

### 创建 ingress

创建 ingress 的代码实现：

```go
		// Create Ingress endpoint
		log.Printf("Creating ingress for app %v ....", m.app.AppName)
		if _, err := m.CreateIngressService(); err != nil {
			return err
		}
		log.Printf("Ingress for app %v has been created.", m.app.AppName)
```

从日志上看，创建 ingress 的速度很快，不到1秒：

```bash
2022/03/27 10:41:09 Creating ingress for app testapp ....
2022/03/27 10:41:10 Ingress for app testapp has been created.

2022/03/27 10:41:19 Creating ingress for app tester ....
2022/03/27 10:41:19 Ingress for app tester has been created.
```

但特别注意：这里的所谓created，应该只是将命令发给了k8s，也就是这里是异步返回。并不是 ingress 立即可用。

### 创建端口转发 

创建 ingress 的代码实现：

```go
		log.Printf("Creating pod port forwarder for app %v ....", m.app.AppName)
		m.forwarder = NewPodPortForwarder(m.client, m.namespace)
		log.Printf("Pod port forwarder for app %v has been created.", m.app.AppName)
```

从日志上看，创建端口转发的速度也非常快，不到1秒：

```bash
2022/03/27 10:41:10 Creating pod port forwarder for app testapp ....
2022/03/27 10:41:10 Pod port forwarder for app testapp has been created.

2022/03/27 10:41:19 Creating pod port forwarder for app tester ....
2022/03/27 10:41:19 Pod port forwarder for app tester has been created.
```

但特别注意：这里的所谓created，应该只是将命令发给了k8s，也就是这里是异步返回。并不是端口转发立即可用。

### 完整的日志分析

下面是一个完整的日志，安装 testapp 和 tester 两个应用，耗时 19秒：

```bash
2022/03/27 11:48:39 Running setup...
2022/03/27 11:48:39 Installing test apps...
2022/03/27 11:48:39 Adding app {testapp 0  map[] true perf-service_invocation_http:dev-linux-amd64  docker.io/skyao 1 true true   4.0 0.1 800Mi 2500Mi 4.0 0.1 512Mi 250Mi <nil> false}
2022/03/27 11:48:39 Adding app {tester 3001  map[] true perf-tester:dev-linux-amd64  docker.io/skyao 1 true true   4.0 0.1 800Mi 2500Mi 4.0 0.1 512Mi 250Mi <nil> false}
2022/03/27 11:48:39 Installing apps ...
2022/03/27 11:48:41 Deploying app testapp ...
2022/03/27 11:48:48 App testapp has been deployed.
2022/03/27 11:48:48 Validating sidecar for app testapp ....
2022/03/27 11:48:48 Streaming Kubernetes logs to ./container_logs/testapp-85d8d9db89-llmrr.daprd.log
2022/03/27 11:48:48 Streaming Kubernetes logs to ./container_logs/testapp-85d8d9db89-llmrr.testapp.log
2022/03/27 11:48:49 Sidecar for app testapp has been validated.
2022/03/27 11:48:49 Creating ingress for app testapp ....
2022/03/27 11:48:49 Ingress for app testapp has been created.
2022/03/27 11:48:49 Creating pod port forwarder for app testapp ....
2022/03/27 11:48:49 Pod port forwarder for app testapp has been created.
2022/03/27 11:48:50 Deploying app tester ...
2022/03/27 11:48:57 App tester has been deployed.
2022/03/27 11:48:57 Validating sidecar for app tester ....
2022/03/27 11:48:57 Streaming Kubernetes logs to ./container_logs/tester-7944b6bb68-wzfj2.tester.log
2022/03/27 11:48:57 Streaming Kubernetes logs to ./container_logs/tester-7944b6bb68-wzfj2.daprd.log
2022/03/27 11:48:58 Sidecar for app tester has been validated.
2022/03/27 11:48:58 Creating ingress for app tester ....
2022/03/27 11:48:58 Ingress for app tester has been created.
2022/03/27 11:48:58 Creating pod port forwarder for app tester ....
2022/03/27 11:48:58 Pod port forwarder for app tester has been created.
2022/03/27 11:48:58 Apps are installed.
2022/03/27 11:48:58 Running tests...
```

这19秒总时长中比较耗时的操作主要是：

- Installing apps： 2 秒
- 部署 testapp： 7秒
- 部署 tester： 7秒
- 验证sidecar/创建ingress和端口转发：1秒 （两个app x 2)



### 等待测试应用就绪的流程

#### 等待流程就绪

由于存在一个安装测试应用的流程， 而 k8s 部署/启动完成测试应用是需要一段时间的，因此，就需要一个机制能等待并检测到测试应用是否安装完成，这样才可以开启真正的测试即开始执行 testcase。

以 TestServiceInvocationHTTPPerformance 为例，删除测试执行的细节代码：

```go
const numHealthChecks = 60 // Number of times to check for endpoint health per app.

func TestServiceInvocationHTTPPerformance(t *testing.T) {
	p := perf.Params()
	t.Logf("running service invocation http test with params: qps=%v, connections=%v, duration=%s, payload size=%v, payload=%v", p.QPS, p.ClientConnections, p.TestDuration, p.PayloadSizeKB, p.Payload) // line 79


	// Get the ingress external url of test app
	testAppURL := tr.Platform.AcquireAppExternalURL("testapp")
	require.NotEmpty(t, testAppURL, "test app external URL must not be empty")

	// Check if test app endpoint is available
	t.Logf("test app url: %s", testAppURL+"/test")   // line 86
	_, err := utils.HTTPGetNTimes(testAppURL+"/test", numHealthChecks)    // 在这里等待测试应用 testapp 就绪！
	require.NoError(t, err)

	// Get the ingress external url of tester app
	testerAppURL := tr.Platform.AcquireAppExternalURL("tester")
	require.NotEmpty(t, testerAppURL, "tester app external URL must not be empty")

	// Check if tester app endpoint is available
	t.Logf("teter app url: %s", testerAppURL)    // line 95							// 在这里等待测试应用 tester 就绪！
	_, err = utils.HTTPGetNTimes(testerAppURL, numHealthChecks)
	require.NoError(t, err)

	// Perform baseline test
	......

	// Perform dapr test
	......
}
```

从执行日志中分别提取对应上面三处的日志内容：

```bash
    service_invocation_http_test.go:79: running service invocation http test with params: qps=1, connections=1, duration=1m, payload size=0, payload=
2022/03/27 11:48:58 Waiting until service ingress is ready for testapp...
2022/03/27 11:49:03 Service ingress for testapp is ready...
    service_invocation_http_test.go:86: test app url: 20.84.11.6:3000/test
2022/03/27 11:49:04 Waiting until service ingress is ready for tester...
2022/03/27 11:49:33 Service ingress for tester is ready...
    service_invocation_http_test.go:95: teter app url: 20.85.250.98:3000
```

这两处就是在等待测试应用 testapp 和 tester 启动完成。检查的方式就是访问这两个应用的 public url （也就是 health check的地址），如果能访问（可连接，返回 htltp 200）则说明应用启动完成。如果失败，则继续等待。

这里面实际是有两个等待：

1. 等待测试应用的 ingress 就绪
2. 等待测试应用自身就绪

##### 等待测试应用的 ingress 就绪

因为 pod ip 不能直接在 k8s 下访问，因此需要通过 ingress 和 端口转发。前面安装测试应用的流程中作了 ingress 的创建和端口转发的创建，但生效是需要时间的。在 AcquireAppExternalURL() 方法中会等待 ingress 就绪：

```bash
	testAppURL := tr.Platform.AcquireAppExternalURL("testapp")
	testerAppURL := tr.Platform.AcquireAppExternalURL("tester")
```

k8s下，实现的代码在 `tests/runner/kube_testplatform.go` 中：

```go
// AcquireAppExternalURL returns the external url for 'name'.
func (c *KubeTestPlatform) AcquireAppExternalURL(name string) string {
	app := c.AppResources.FindActiveResource(name)
	return app.(*kube.AppManager).AcquireExternalURL()
}

// AcquireExternalURL gets external ingress endpoint from service when it is ready.
func (m *AppManager) AcquireExternalURL() string {
	log.Printf("Waiting until service ingress is ready for %s...\n", m.app.AppName)
	svc, err := m.WaitUntilServiceState(m.IsServiceIngressReady) // 等待直到 ingress reday
	if err != nil {
		return ""
	}

	log.Printf("Service ingress for %s is ready...\n", m.app.AppName)
	return m.AcquireExternalURLFromService(svc)
}
```

具体的等待实现在 WaitUntilServiceState() 方法中：

```go
	// PollInterval is how frequently e2e tests will poll for updates.
	PollInterval = 1 * time.Second
	// PollTimeout is how long e2e tests will wait for resource updates when polling.
	PollTimeout = 10 * time.Minute
	
	// WaitUntilServiceState waits until isState returns true.
func (m *AppManager) WaitUntilServiceState(isState func(*apiv1.Service, error) bool) (*apiv1.Service, error) {
	serviceClient := m.client.Services(m.namespace)
	var lastService *apiv1.Service

	waitErr := wait.PollImmediate(PollInterval, PollTimeout, func() (bool, error) {
		var err error
		lastService, err = serviceClient.Get(context.TODO(), m.app.AppName, metav1.GetOptions{})
		done := isState(lastService, err)
		if !done && err != nil {
			return true, err
		}

		return done, nil
	})

	......
	return lastService, nil
}
```

每隔1秒，时长 10 分钟，这个时间长度有点离谱。之前遇到过 ingress 无法访问的案例，就在这里等待长达10 分钟。

但如果能正常工作，只是启动速度慢，那这里的10分钟怎么也够 ingress 生效的。

##### 等待测试应用自身就绪

实现代码如下：

```go
const numHealthChecks = 60 // Number of times to check for endpoint health per app.

// HTTPGetNTimes calls the url n times and returns the first success or last error.
func HTTPGetNTimes(url string, n int) ([]byte, error) {
	var res []byte
	var err error
	for i := n - 1; i >= 0; i-- {
		res, err = HTTPGet(url)
		if i == 0 {
			break
		}

		if err != nil {
			time.Sleep(time.Second)
		} else {
			return res, nil
		}
	}

	return res, err
}
```

每秒检查一次，重试 60 次。60 秒之后只要启动成功就可以。

>  备注：这里的检查顺序，先检查 testapp，再检查 tester app，所以如果两者都启动的慢的话，顺序偏后的 tester app 有更多的启动时间。

### bug：总是卡在等待测试应用就绪上

测试中发现，测试案例总是卡在等待测试应用就绪上，但奇怪的是，从打印日志上看，上面的两个等待过程中，第一个等待 ingress 达到 ready 状态总是通过，然后等待测试应用自身就绪（也就是通过 http://52.226.222.31:3000/test 地址访问）就总是不能成功：

```bash
=== Failed
=== FAIL: tests/perf/service_invocation_http TestServiceInvocationHTTPPerformance (248.07s)
    service_invocation_http_test.go:79: running service invocation http test with params: qps=1, connections=1, duration=1m, payload size=0, payload=
2022/03/25 21:49:17 Waiting until service ingress is ready for testapp...
2022/03/25 21:49:20 Service ingress for testapp is ready...
    service_invocation_http_test.go:86: test app url: 52.226.222.31:3000/test
    service_invocation_http_test.go:88: 
                Error Trace:    service_invocation_http_test.go:88
                Error:          Received unexpected error:
                                Get "http://52.226.222.31:3000/test": EOF
                Test:           TestServiceInvocationHTTPPerformance

```

翻了一下 `tests/apps/perf/service_invocation_http/app.go` 的代码，这是服务器端appt的实现，超级简单：

```go
func handler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(200)
}

func main() {
	http.HandleFunc("/test", handler)
	log.Fatal(http.ListenAndServe(":3000", nil))
}
```

从 `k get pods -n dapr-tests` 命令的输出看，testapp 的状态很早就是 running了。这个简单的应用不存在 60 秒还启动不起来的情况。但奇怪的是，有时这个问题又不存在，能正常的访问。而且一旦正常就会一直都正常，一旦不正常就一直不正常。

经过反复测试排查发现：在使用 azure 部署 k8s 时，pod 的外部访问地址，必须在连接公司 VPN （GlobalProtect）时才能正常访问，如果 VPN 未能开启，则会报错 Empty reply from server ，一直卡在这里直到 60 秒超时。

```bash
# 断开vpn
$ curl -i 20.81.110.42:3000/test
curl: (52) Empty reply from server

# 连接vpn
$ curl -i 20.81.110.42:3000/test
HTTP/1.1 200 OK
Date: Sun, 27 Mar 2022 07:10:02 GMT
Content-Length: 0

# 再次断开vpn
$ curl -i 20.81.110.42:3000/test
curl: (52) Empty reply from server

# 再次连接vpn
$ curl -i 20.81.110.42:3000/test
HTTP/1.1 200 OK
Date: Sun, 27 Mar 2022 07:12:05 GMT
Content-Length: 0
```

排除这个问题之后，性能测试的案例就可以正常

> 备注：我是在我本地机器 macbook 上跑测试案例的，案例和 azure 上的k8s集群通讯一直正常，但是就是有时可以访问 app 的external url，有时不能。没想到是这个原因



