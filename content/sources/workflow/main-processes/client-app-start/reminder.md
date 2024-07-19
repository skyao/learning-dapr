---
title: "reminder实现流程"
linkTitle: "reminder"
weight: 30
date: 2021-02-24
description: >
  client app start流程中reminder实现的源码分析
---

## 背景

## new reminder



## reminder trigger

reminder 是如何触发的


## reminder execute

reminder 的执行代码在 `pkg/actors/actors.go` 中的 doExecuteReminderOrTimer() 方法

```go
// Executes a reminder or timer
func (a *actorsRuntime) doExecuteReminderOrTimer(ctx context.Context, reminder *internal.Reminder, isTimer bool) (err error) {
	var (
		data         any
		logName      string
		invokeMethod string
	)

	fmt.Printf("**** versioning ****: doExecuteReminderOrTimer\n")


	// Sanity check: make sure the actor is actually locally-hosted
	isLocal, _ := a.isActorLocallyHosted(ctx, reminder.ActorType, reminder.ActorID)
	if !isLocal {
		return errors.New("actor is not locally hosted")
	}

	if isTimer {
		logName = "timer"
		invokeMethod = "timer/" + reminder.Name
		data = &TimerResponse{
			Callback: reminder.Callback,
			Data:     reminder.Data,
			DueTime:  reminder.DueTime,
			Period:   reminder.Period.String(),
		}
	} else {
		logName = "reminder"
		invokeMethod = "remind/" + reminder.Name
		data = &ReminderResponse{
			DueTime: reminder.DueTime,
			Period:  reminder.Period.String(),
			Data:    reminder.Data,
		}
	}
	policyDef := a.resiliency.ActorPreLockPolicy(reminder.ActorType, reminder.ActorID)

	log.Debug("Executing " + logName + " for actor " + reminder.Key())
	req := invokev1.NewInvokeMethodRequest(invokeMethod).
		WithActorRevision(reminder.ActorType, reminder.ActorID, 77).
		WithDataObject(data).
		WithContentType(invokev1.JSONContentType)
	if policyDef != nil {
		req.WithReplay(policyDef.HasRetries())
	}
	defer req.Close()

	policyRunner := resiliency.NewRunnerWithOptions(ctx, policyDef,
		resiliency.RunnerOpts[*invokev1.InvokeMethodResponse]{
			Disposer: resiliency.DisposerCloser[*invokev1.InvokeMethodResponse],
		},
	)
	imr, err := policyRunner(func(ctx context.Context) (*invokev1.InvokeMethodResponse, error) {
		return a.callLocalActor(ctx, req)
	})
	if err != nil && !errors.Is(err, internal.ErrReminderCanceled) {
		log.Errorf("Error executing %s for actor %s: %v", logName, reminder.Key(), err)
	}
	if imr != nil {
		_ = imr.Close()
	}
	return err
}
```