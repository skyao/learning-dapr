---
title: "如何使用fortio实现性能测试"
linkTitle: "如何使用fortio"
weight: 20
date: 2021-05-09
description: >
  Dapr的性能测试工具是如何使用fortio的
---

## fortio介绍

Dapr 采用 fortio 作为性能测试工具。

Fortio 相关内容已经转移到单独的笔记中：[Learning Fortio](https://skyao.io/learning-fortio/)

## tester 应用

### tester 镜像的生成方法

tester 应用的镜像生成由三个镜像组成：

#### 构建 tester go app 二进制文件的镜像

```dockerfile
FROM golang:1.17 as build_env

ARG GOARCH_ARG=amd64

WORKDIR /app
COPY app.go go.mod ./
RUN go get -d -v && GOOS=linux GOARCH=$GOARCH_ARG go build -o tester .
```

这个 dockerfile 会将 dapr 仓库下 `tests/apps/perf/tester` 目录中的 app.go 和 go.mod 文件复制到镜像中，然后执行 go get 和 go build 命令将 go 代码打包为名为 tester 的二进制可执行文件。

最终的产出物是 `/app/tester` 这个二进制可执行文件。

#### 构建 fortio 二进制文件的镜像

```dockerfile
FROM golang:1.17 as fortio_build_env

ARG GOARCH_ARG=amd64

WORKDIR /fortio
ADD "https://api.github.com/repos/fortio/fortio/branches/master" skipcache
RUN git clone https://github.com/fortio/fortio.git
RUN cd fortio && git checkout v1.16.1 && GOOS=linux GOARCH=$GOARCH_ARG go build
```

这个镜像是构建 fortio v1.16.1 的代码。

最终的产出物是 `/fortio/fortio/fortio` 这个二进制可执行文件。

#### 构建 buster-slim 的镜像

```dockerfile
FROM debian:buster-slim
#RUN apt update
#RUN apt install wget -y
WORKDIR /
COPY --from=build_env /app/tester /
COPY --from=fortio_build_env /fortio/fortio/fortio /usr/local/bin
CMD ["/tester"]
```

这个就只是复制前两个镜像的产出物了，将 `/app/tester` 复制到根目录，将 fortio 复制到 

`/usr/local/bin` 目录。

## tester 应用的工作原理和实现代码

`tests/apps/perf/tester/app.go` 的核心代码如下：

### main 函数

```go
func main() {
	http.HandleFunc("/", handler)
	http.HandleFunc("/test", testHandler)
	log.Fatal(http.ListenAndServe(":3001", nil))
}
```

main 函数启动 http server，监听 3001 端口，然后注册了两个路径和对应的 handler。

### 简单探活 handler

这个 handler 超级简单，什么都不做，只是返回 http 200 。

```go
func main() {
	http.HandleFunc("/", handler)
  ......
}

func handler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(200)
}
```

### 测试handler

testHandler 执行性能测试：

```go
func main() {
	http.HandleFunc("/test", testHandler)
  ......
}

func testHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Println("test execution request received")

  // 步骤1: 从请求中读取测试相关的配置参数，这些参数是从 test case 中发出的
	var testParams TestParameters
	b, err := io.ReadAll(r.Body)
	if err != nil {
		w.WriteHeader(500)
		w.Write([]byte(fmt.Sprintf("error reading request body: %s", err)))
		return
	}

  // 步骤2: 解析读取的测试相关的配置参数
	err = json.Unmarshal(b, &testParams)
	if err != nil {
		w.WriteHeader(400)
		w.Write([]byte(fmt.Sprintf("error parsing test params: %s", err)))
		return
	}

  // 步骤3: 开始执行性能测试
	fmt.Println("executing test")
	results, err := runTest(testParams)
	if err != nil {
		w.WriteHeader(500)
		w.Write([]byte(fmt.Sprintf("error encountered while running test: %s", err)))
		return
	}

  // 步骤4: 返回性能测试的结果
	fmt.Println("test finished")
	w.Header().Add("Content-Type", "application/json")
	w.Write(results)
}
```

### 真正的性能测试

真正的性能测试是通过 exec.Command 来执行命令行，通过调用 fortio 工具来进行的，也即是说，前面的 tester 应用除了用来启动 daprd 外，tester 自身只是配合走完性能测试的流程，真正的性能测试是由 fortio 进行。

```go
// runTest accepts a set of test parameters, runs Fortio with the configured setting and returns
// the test results in json format.
func runTest(params TestParameters) ([]byte, error) {
	var args []string

  // 步骤1: 根据请求参数构建不同的 fortio 执行参数
	if len(params.Payload) > 0 {
		args = []string{
			"load", "-json", "result.json", "-content-type", "application/json", "-qps", fmt.Sprint(params.QPS), "-c", fmt.Sprint(params.ClientConnections),
			"-t", params.TestDuration, "-payload", params.Payload,
		}
	} else {
		args = []string{
			"load", "-json", "result.json", "-qps", fmt.Sprint(params.QPS), "-c", fmt.Sprint(params.ClientConnections),
			"-t", params.TestDuration, "-payload-size", fmt.Sprint(params.PayloadSizeKB),
		}
	}
	if params.StdClient {
		args = append(args, "-stdclient")
	}
	args = append(args, params.TargetEndpoint)
	fmt.Printf("running test with params: %s", args)

  // 步骤2: 调用 fortio 执行性能测试
	cmd := exec.Command("fortio", args...)
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	err := cmd.Run()
	if err != nil {
		return nil, err
	}
  
  // 步骤3: 返回性能测试执行结果
	return os.ReadFile("result.json")
}
```

#### 无负载的性能测试

对于 payload 大小为 0 的情况，执行的是如下的 fortio 命令：

```bash
fortio load -json result.json -qps ${QPS} -c ${ClientConnections} -t ${TestDuration} -payload-size ${PayloadSizeKB} ${TargetEndpoint}
```

> 疑问：payload 都为零了，为啥了还要设置 -payload-size ？

翻了一下相关的日志，找到对应的日志内容为：

```bash
fmt.Printf("running test with params: %s", args)

running test with params: [load -json result.json -qps 1 -c 1 -t 1m -payload-size 0 http://testapp:3000/test]
```

#### 带负载的性能测试

对于 payload 大小不为 0 的情况，执行的是如下的 fortio 命令：

```bash
fortio load -json result.json -content-type application/json -qps ${QPS} -c ${ClientConnections} -t ${TestDuration} -payload ${Payload} ${TargetEndpoint}
```

## 全流程参数传递

在 perf test 的测试过程中，参数传递比较复杂（或者说比较绕），期间涉及到 perf test case 如何

```plantuml
title parameters transfer in load test
hide footbox
skinparam style strictuml

actor test_case as "Test Case"
participant tester as "Tester App"
box "Fortio" #LightBlue
participant fortio_load as "fortioLoad"
participant fgrpc
participant dapr as "daprResults"
end box
box "daprd"
participant grpc_api as "gRPC API"
end box

test_case -> tester : HTTP Post
note left: TestParameters
tester -> fortio_load : exec
note left: fortio load ... -grpc -dapr k1=v1,k2=v2
fortio_load -> fgrpc : RunGRPCTest()
note left: -grpc -dapr k1=v1,k2=v2
fgrpc -> dapr : RunTest()
note left: -dapr k1=v1,k2=v2
dapr -> grpc_api
dapr <-- grpc_api
fgrpc <-- dapr
fortio_load <-- fgrpc
tester <-- fortio_load
test_case <-- tester
```

### 步骤1：perf test case 获取测试参数

perf test case 在启动时，会从环境变量中获取测试参数

- DAPR_PERF_QPS: 默认1
- DAPR_PERF_CONNECTIONS： 默认1
- DAPR_TEST_DURATION： 默认 "1m"，即1分钟
- DAPR_PAYLOAD_SIZE: 默认0
- DAPR_PAYLOAD: 默认为空(字符串"")

### 步骤2：perf test case打包测试参数发送给tester app

perf test case 会发送一个 HTTP 请求到 tester app，其内容如下：

- URL： http://testerAppURL/test

- Method: POST

- Body:  将 TestParameters 结构体系列化为 json 

TestParameters 结构体的字段如下所示：

```go
type TestParameters struct {
	QPS               int    `json:"qps"`
	ClientConnections int    `json:"clientConnections"`
	TargetEndpoint    string `json:"targetEndpoint"`
	TestDuration      string `json:"testDuration"`
	PayloadSizeKB     int    `json:"payloadSizeKB"`
	Payload           string `json:"payload"`
	StdClient         bool   `json:"stdClient"`
}
```

在支持 gRPC + dapr 时，由于没有合适的参数可以使用，因此增加了 grpc 和 dapr 两个字段：

```go
type TestParameters struct {
	......
	Grpc              bool   `json:"grpc"`
	Dapr              string `json:"dapr"`
}
```

### 步骤3：tester app解析测试参数，传递给fortio

在 tester app (`tests/apps/perf/tester/app.go`) 的 testHandler() 方法中，会对传入http body进行读取和json解析，然后将 TestParameters 转为 fortio 的参数：

```go
func testHandler(w http.ResponseWriter, r *http.Request) {
	b, err := io.ReadAll(r.Body)
	err = json.Unmarshal(b, &testParams)
	results, err := runTest(testParams)
    ......
}
func runTest(params TestParameters) ([]byte, error) {
	args := buildFortioArgs(params)
    ......
}
```

TestParameters 字段和 fortio 参数的对应关系：


| 语义           | 环境变量              | TestParameters 字段 | fortio 参数                      |
| -------------- | --------------------- | ------------------- | -------------------------------- |
|                |                       |                     | load                             |
|                |                       |                     | `-json result.json`              |
|                |                       |                     | `-content-type application/json` |
| QPS            | DAPR_PERF_QPS         | QPS                 | `-qps`                           |
| 客户端连接数   | DAPR_PERF_CONNECTIONS | ClientConnections   | `-c`                             |
| 测试时长       | DAPR_TEST_DURATION    | TestDuration        | `-t`                             |
| 负载           | DAPR_PAYLOAD          | Payload             | `-payload`                       |
| 负载大小       | DAPR_PAYLOAD_SIZE     | PayloadSizeKB       | `-payload-size`                  |
|                |                       | StdClient           | `-stdclient`                     |
| 是否是grpc测试 |                       | Grpc                | `-grpc`                          |
| Dapr测试参数   |                       | Dapr                | `-dapr`                          |

### 步骤4：fortio解析 dapr flag，在发起的dapr 请求中使用

在 fortio 的执行中， `load` 子命令会有 `fortioLoad` 方法负责，检查发现有 `-grpc` flag，则会转到 `fgrpc.RunGRPCTest()` 方法。

fortio 的 grpc 支持默认只有自带的 ping 和标准的 health，为了支持 dapr ，我们扩展了 `fgrpc.RunGRPCTest()` 方法。

`-dapr` flag 传递的参数会被透传到  dapr 的扩展代码中， 然后解析为下面的结构：

```go
type DaprRequestParameters struct {
	capability string
	target     string
	method     string
	appId      string
	store      string

	extensions map[string]string
}
```

这些参数将在后面 fortio 扩展代码中进行 dapr 调用时被使用到。
