---
title: "执行orchestrator task"
linkTitle: "执行orchestrator task"
weight: 30
date: 2021-02-24
description: >
  workflow runtime运行时执行orchestrator task的源码
---

## 回顾

前面看到执行orchestrator task的代码实现在 durabletask-go 仓库的 `client/src/main/java/com/microsoft/durabletask/DurableTaskGrpcWorker.java` 中。

```java
TaskOrchestrationExecutor taskOrchestrationExecutor = new TaskOrchestrationExecutor(
        this.orchestrationFactories,
        this.dataConverter,
        this.maximumTimerInterval,
        logger);
......
Iterator<WorkItem> workItemStream = this.sidecarClient.getWorkItems(getWorkItemsRequest);
while (workItemStream.hasNext()) {
    WorkItem workItem = workItemStream.next();
    RequestCase requestType = workItem.getRequestCase();
    if (requestType == RequestCase.ORCHESTRATORREQUEST) {
        OrchestratorRequest orchestratorRequest = workItem.getOrchestratorRequest();

        TaskOrchestratorResult taskOrchestratorResult = taskOrchestrationExecutor.execute(
                orchestratorRequest.getPastEventsList(),
                orchestratorRequest.getNewEventsList());

        OrchestratorResponse response = OrchestratorResponse.newBuilder()
                .setInstanceId(orchestratorRequest.getInstanceId())
                .addAllActions(taskOrchestratorResult.getActions())
                .setCustomStatus(StringValue.of(taskOrchestratorResult.getCustomStatus()))
                .build();

        this.sidecarClient.completeOrchestratorTask(response);
    }
    ......
```

## 实现细节

### TaskOrchestrationExecutor

TaskOrchestrationExecutor 类的定义和构造函数：

```java
 final class TaskOrchestrationExecutor {

    private static final String EMPTY_STRING = "";
    private final HashMap<String, TaskOrchestrationFactory> orchestrationFactories;
    private final DataConverter dataConverter;
    private final Logger logger;
    private final Duration maximumTimerInterval;

    public TaskOrchestrationExecutor(
            HashMap<String, TaskOrchestrationFactory> orchestrationFactories,
            DataConverter dataConverter,
            Duration maximumTimerInterval,
            Logger logger) {
        this.orchestrationFactories = orchestrationFactories;
        this.dataConverter = dataConverter;
        this.maximumTimerInterval = maximumTimerInterval;
        this.logger = logger;
    }
```

其中 orchestrationFactories 是从前面 registerWorkflow()时保存的已经注册的工作流信息。

execute() 方法：

```java
public TaskOrchestratorResult execute(List<HistoryEvent> pastEvents, List<HistoryEvent> newEvents) {
    ContextImplTask context = new ContextImplTask(pastEvents, newEvents);

    boolean completed = false;
    try {
        // Play through the history events until either we've played through everything
        // or we receive a yield signal
        while (context.processNextEvent()) { /* no method body */ }
        completed = true;
    } catch (OrchestratorBlockedException orchestratorBlockedException) {
        logger.fine("The orchestrator has yielded and will await for new events.");
    } catch (ContinueAsNewInterruption continueAsNewInterruption) {
        logger.fine("The orchestrator has continued as new.");
        context.complete(null);
    } catch (Exception e) {
        // The orchestrator threw an unhandled exception - fail it
        // TODO: What's the right way to log this?
        logger.warning("The orchestrator failed with an unhandled exception: " + e.toString());
        context.fail(new FailureDetails(e));
    }

    if ((context.continuedAsNew && !context.isComplete) || (completed && context.pendingActions.isEmpty() && !context.waitingForEvents())) {
        // There are no further actions for the orchestrator to take so auto-complete the orchestration.
        context.complete(null);
    }

    return new TaskOrchestratorResult(context.pendingActions.values(), context.getCustomStatus());
}
```

这里只是主要流程，细节实现在内部私有类 ContextImplTask 中。

#### ContextImplTask

ContextImplTask 的定义和构造函数，使用到 OrchestrationHistoryIterator。

```java
private class ContextImplTask implements TaskOrchestrationContext {
    private final OrchestrationHistoryIterator historyEventPlayer;
    ......

    public ContextImplTask(List<HistoryEvent> pastEvents, List<HistoryEvent> newEvents) {
        this.historyEventPlayer = new OrchestrationHistoryIterator(pastEvents, newEvents);
    }
    ......

    private boolean processNextEvent() {
        return this.historyEventPlayer.moveNext();
    }
}
```

#### OrchestrationHistoryIterator

OrchestrationHistoryIterator 的类定义和构造函数，其中 pastEvents 和 newEvents 是 daprd sidecar 那边在 getWorkItem() 返回的 orchestratorRequest 中携带的数据。

```java
private class OrchestrationHistoryIterator {
    private final List<HistoryEvent> pastEvents;
    private final List<HistoryEvent> newEvents;

    private List<HistoryEvent> currentHistoryList;
    private int currentHistoryIndex;

    public OrchestrationHistoryIterator(List<HistoryEvent> pastEvents, List<HistoryEvent> newEvents) {
        this.pastEvents = pastEvents;
        this.newEvents = newEvents;
        this.currentHistoryList = pastEvents;
    }
```

currentHistoryList 初始化指向 pastEvents，currentHistoryIndex 为0。

然后继续看 moveNext() 方法：

```java
    public boolean moveNext() {
        if (this.currentHistoryList == pastEvents && this.currentHistoryIndex >= pastEvents.size()) {
            // 如果当前 currentHistoryList 指向的是 pastEvents，并且已经指到最后一个元素了。
            // 那么 moveNext 就应该指向 this.newEvents，然后将 currentHistoryIndex 设置为0 （即指向第一个元素）
            // Move forward to the next list
            this.currentHistoryList = this.newEvents;
            this.currentHistoryIndex = 0;

            // 这意味着 pastEvents 的游历接触，即 replay 完成。
            ContextImplTask.this.setDoneReplaying();
        }

        if (this.currentHistoryList == this.newEvents && this.currentHistoryIndex >= this.newEvents.size()) {
            // 如果当前 currentHistoryList 指向的是 newEvents，并且已经指到最后一个元素了。
            // 此时已经完成游历，没有更多元素，返回 false 表示可以结束了。
            // We're done enumerating the history
            return false;
        }

        // Process the next event in the history
        // 获取当前元素，然后 currentHistoryIndex +1 指向下一个元素
        HistoryEvent next = this.currentHistoryList.get(this.currentHistoryIndex++);
        // 处理事件
        ContextImplTask.this.processEvent(next);
        return true;
    }
```

处理事件的代码实现在 ContextImplTask 的 processEvent() 方法中：

```java
        private void processEvent(HistoryEvent e) {
            boolean overrideSuspension = e.getEventTypeCase() == HistoryEvent.EventTypeCase.EXECUTIONRESUMED || e.getEventTypeCase() == HistoryEvent.EventTypeCase.EXECUTIONTERMINATED;
            if (this.isSuspended && !overrideSuspension) {
                this.handleEventWhileSuspended(e);
            } else {
                switch (e.getEventTypeCase()) {
                    case ORCHESTRATORSTARTED:
                        Instant instant = DataConverter.getInstantFromTimestamp(e.getTimestamp());
                        this.setCurrentInstant(instant);
                        break;
                    case ORCHESTRATORCOMPLETED:
                        // No action
                        break;
                    case EXECUTIONSTARTED:
                        ExecutionStartedEvent startedEvent = e.getExecutionStarted();
                        String name = startedEvent.getName();
                        this.setName(name);
                        String instanceId = startedEvent.getOrchestrationInstance().getInstanceId();
                        this.setInstanceId(instanceId);
                        String input = startedEvent.getInput().getValue();
                        this.setInput(input);
                        TaskOrchestrationFactory factory = TaskOrchestrationExecutor.this.orchestrationFactories.get(name);
                        if (factory == null) {
                            // Try getting the default orchestrator
                            factory = TaskOrchestrationExecutor.this.orchestrationFactories.get("*");
                        }
                        // TODO: Throw if the factory is null (orchestration by that name doesn't exist)
                        TaskOrchestration orchestrator = factory.create();
                        orchestrator.run(this);
                        break;
//                case EXECUTIONCOMPLETED:
//                    break;
//                case EXECUTIONFAILED:
//                    break;
                    case EXECUTIONTERMINATED:
                        this.handleExecutionTerminated(e);
                        break;
                    case TASKSCHEDULED:
                        this.handleTaskScheduled(e);
                        break;
                    case TASKCOMPLETED:
                        this.handleTaskCompleted(e);
                        break;
                    case TASKFAILED:
                        this.handleTaskFailed(e);
                        break;
                    case TIMERCREATED:
                        this.handleTimerCreated(e);
                        break;
                    case TIMERFIRED:
                        this.handleTimerFired(e);
                        break;
                    case SUBORCHESTRATIONINSTANCECREATED:
                        this.handleSubOrchestrationCreated(e);
                        break;
                    case SUBORCHESTRATIONINSTANCECOMPLETED:
                        this.handleSubOrchestrationCompleted(e);
                        break;
                    case SUBORCHESTRATIONINSTANCEFAILED:
                        this.handleSubOrchestrationFailed(e);
                        break;
//                case EVENTSENT:
//                    break;
                    case EVENTRAISED:
                        this.handleEventRaised(e);
                        break;
//                case GENERICEVENT:
//                    break;
//                case HISTORYSTATE:
//                    break;
//                case EVENTTYPE_NOT_SET:
//                    break;
                    case EXECUTIONSUSPENDED:
                        this.handleExecutionSuspended(e);
                        break;
                    case EXECUTIONRESUMED:
                        this.handleExecutionResumed(e);
                        break;
                    default:
                        throw new IllegalStateException("Don't know how to handle history type " + e.getEventTypeCase());
                }
            }
        }
```

这里具体会执行什么代码，就要看给过来的 event 是什么了。

### EXECUTIONSTARTED 事件的执行

这是 ExecutionStartedEvent 的 proto 定义：

```protobuf
message ExecutionStartedEvent {
    string name = 1;
    google.protobuf.StringValue version = 2;
    google.protobuf.StringValue input = 3;
    OrchestrationInstance orchestrationInstance = 4;
    ParentInstanceInfo parentInstance = 5;
    google.protobuf.Timestamp scheduledStartTimestamp = 6;
    TraceContext parentTraceContext = 7;
    google.protobuf.StringValue orchestrationSpanID = 8;
}
```

EXECUTIONSTARTED 事件的处理：

```java
case EXECUTIONSTARTED:
    ExecutionStartedEvent startedEvent = e.getExecutionStarted();
    String name = startedEvent.getName();
    this.setName(name);
    String instanceId = startedEvent.getOrchestrationInstance().getInstanceId();
    this.setInstanceId(instanceId);
    String input = startedEvent.getInput().getValue();
    this.setInput(input);
    TaskOrchestrationFactory factory = TaskOrchestrationExecutor.this.orchestrationFactories.get(name);
    if (factory == null) {
        // Try getting the default orchestrator
        factory = TaskOrchestrationExecutor.this.orchestrationFactories.get("*");
    }
    // TODO: Throw if the factory is null (orchestration by that name doesn't exist)
    TaskOrchestration orchestrator = factory.create();
    orchestrator.run(this);
    break;
```

name / instanceId / input 等基本信息直接设置在 ContextImplTask 上。

factory 要从 orchestrationFactories 里面根据 name 查找，如果没有找到，则获取默认。

从 factory 创建 TaskOrchestration，再运行 orchestrator.run()：

```java
    TaskOrchestration orchestrator = factory.create();
    orchestrator.run(this);
```

这就回到 TaskOrchestration 的实现了。

### OrchestratorWrapper

Dapr java sdk 中的 OrchestratorWrapper 实现了 TaskOrchestration 接口

```java
class OrchestratorWrapper<T extends Workflow> implements TaskOrchestrationFactory {
  @Override
  public TaskOrchestration create() {
    return ctx -> {
      T workflow;
      try {
        workflow = this.workflowConstructor.newInstance();
      } ......
    };
  }
}
```



