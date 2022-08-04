---
title: "controller"
linkTitle: "controller"
weight: 20
date: 2022-07-24
description: >
  处理 dapr callback 请求的 springboot controller
---



```java
@RestController
public class DaprController {
}
```

### healthz endpoint

用于 health check 的 endpoint，路径为 "/healthz"，实现为空。

```java
@GetMapping(path = "/healthz")
public void healthz() {
}
```

TBD：这里是否要考虑 sidecar 的某些状态？目前这是只要 sidecar 进程和端口可以访问就会应答状态OK，而不管sidecar 中的功能是否正常。

### dapr configuration endpoint

用于获取 dapr sidecar 的自身配置, 路径为 "/dapr/config"

```java
@GetMapping(path = "/dapr/config", produces = MediaType.APPLICATION_JSON_VALUE)
public byte[] daprConfig() throws IOException {
  return ActorRuntime.getInstance().serializeConfig();
}
```

但看 ActorRuntime 的代码实现，这个 config 是指 actor configuration：

```java
  public byte[] serializeConfig() throws IOException {
    return INTERNAL_SERIALIZER.serialize(this.config);
  }

  private ActorRuntime(ManagedChannel channel, DaprClient daprClient) throws IllegalStateException {
    this.config = new ActorRuntimeConfig();
  }
```

### subscribe endpoint

用于获取当前 dapr sidecar 的 pub/sub 订阅信息，路径为 "/dapr/subscribe":

```java
@GetMapping(path = "/dapr/subscribe", produces = MediaType.APPLICATION_JSON_VALUE)
public byte[] daprSubscribe() throws IOException {
  return SERIALIZER.serialize(DaprRuntime.getInstance().listSubscribedTopics());
}
```

### actor endpoint

用于 actor 的 endpoint，包括 deactive, invoke actor method, invoke actor timer 和 invoke actor reminder:

```java
@DeleteMapping(path = "/actors/{type}/{id}")
  public Mono<Void> deactivateActor(@PathVariable("type") String type,
                                    @PathVariable("id") String id) {
    return ActorRuntime.getInstance().deactivate(type, id);
  }

  @PutMapping(path = "/actors/{type}/{id}/method/{method}")
  public Mono<byte[]> invokeActorMethod(@PathVariable("type") String type,
                                        @PathVariable("id") String id,
                                        @PathVariable("method") String method,
                                        @RequestBody(required = false) byte[] body) {
    return ActorRuntime.getInstance().invoke(type, id, method, body);
  }

  @PutMapping(path = "/actors/{type}/{id}/method/timer/{timer}")
  public Mono<Void> invokeActorTimer(@PathVariable("type") String type,
                                     @PathVariable("id") String id,
                                     @PathVariable("timer") String timer,
                                     @RequestBody byte[] body) {
    return ActorRuntime.getInstance().invokeTimer(type, id, timer, body);
  }

  @PutMapping(path = "/actors/{type}/{id}/method/remind/{reminder}")
  public Mono<Void> invokeActorReminder(@PathVariable("type") String type,
                                        @PathVariable("id") String id,
                                        @PathVariable("reminder") String reminder,
                                        @RequestBody(required = false) byte[] body) {
    return ActorRuntime.getInstance().invokeReminder(type, id, reminder, body);
  }
```