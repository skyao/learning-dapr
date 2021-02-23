---
type: blog
date: 2021-02-18
title: "Dapr API的未来计划"
linkTitle: "Dapr API的未来计划"
description: >
  Dapr API的未来计划，考虑提取为标准规范
---

Dapr API的未来计划，考虑提取为标准规范

## Proposal信息

[[Discussion] Future plans for dapr api #2817](https://github.com/dapr/dapr/issues/2817)

### 提案内容

Firstly, I'd like to congratulate dapr community on the [1.0.0](https://github.com/dapr/dapr/releases/tag/v1.0.0) release, this is a remarkable milestone!

I understand the community was busy with the 1.0.0 release recently, but right now might be a good time to take a cup of coffee and think about the future.!

What I'm interested in and like to discuss here is our future plans for [dapr api](https://github.com/dapr/dapr/tree/master/dapr/proto): ***Do we have the intention to make dapr api a standard so that other sidecars/proxies could implement?***

I've been playing with the dapr demos recently, and one thing that I'm really fond of is its possibility to truly realize **Write once, Run anywhere**, as dapr defines a very good abstraction layer between application code and the backend services. If every application could be sided with a dapr process/sidecar, then this would easily happen.

> 首先，我要祝贺dapr社区发布了1.0.0版本，这是一个了不起的里程碑！
>
> 我知道社区最近在忙于1.0.0版本的发布，但现在也许是喝杯咖啡思考一下未来的好时机。
>
> 我对dapr api的未来计划很感兴趣，也想在这里讨论一下。我们是否打算让dapr api成为一个标准，让其他的sidecars/proxies也能实现？
>
> 最近我一直在用dapr的demo，有一点我非常喜欢，就是它可以真正实现 "一次编写，随处运行"，因为dapr在应用代码和后端服务之间定义了一个非常好的抽象层。如果每个应用都能配上dapr进程/sidecar，那么这一切就很容易实现了。

![](https://user-images.githubusercontent.com/837658/108360923-cb4ed200-722c-11eb-981f-0b7c3582cdd9.png)

However, considering the real world cases: where we have different cloud providers, different legacy components and many other factors I could not possibly list, it would be very hard to make dapr available in every situation.

> 然而，考虑到现实世界中的案例：我们有不同的云提供商，不同的传统组件和许多其他我不可能列出的因素，这将很难使dapr在每种情况下都可用。

So I'm thinking why not we take a step forward and make the abstraction layer a standard? So that other cloud providers could choose to offer hosted dapr solution or their own solutions with the same api, and legacy components could also bridge to the same api. Then as an end user, we could safely program our application against the standard api and ship it to different environments with no code change.

> 所以我在想，我们为什么不往前走一步，把抽象层变成一个标准呢？这样，其他云提供商可以选择提供托管的dapr解决方案或他们自己的解决方案，并使用相同的api，传统组件也可以桥接到相同的api。然后，作为最终用户，我们可以安全地根据标准的api对我们的应用程序进行编程，并在不改变代码的情况下将其发布到不同的环境中。

![](https://user-images.githubusercontent.com/837658/108361024-ed485480-722c-11eb-975b-0757bb09da1f.png)

I guess someone might raise the hand and say this sounds good but it looks like we don't need to do anything as the apis are already defined in the proto files. But what I'm thinking is if we do have the intention to make dapr api a standard, then it would be better to send more obvious signals(e.g. move proto files to a separate repo) or state our intention more clearly so that other sidecars/proxies could implement the dapr apis without concern, e.g. concerns of breaking changes, etc.

Looking forward to your ideas/comments on this topic!

> 我猜有人会举手说，这听起来不错，但看起来我们不需要做任何事情，因为api已经在proto文件中定义了。但我想的是，如果我们真的打算让dapr api成为一个标准，那么最好发出更明显的信号（例如，将proto文件移到一个单独的repo中），或者更清楚地说明我们的意图，这样其他的sidecars/proxies就可以实现dapr apis，而不用担心，例如，担心破坏变化，等等。
> 
> 期待您对这个话题的想法/评论!

### 提案讨论

Hey @nobodyiam thanks for bringing this up. Speaking for myself, I think it makes sense to make the (some?) Dapr APIs a standard as it'll enable interoperability between other technologies.

> Hey @nobodyiam 谢谢你提出这个问题。就我个人而言，我认为将Dapr APIs作为标准是有意义的，因为它可以实现其他技术之间的互操作性。

> I guess someone might raise the hand and say this sounds good but it looks like we don't need to do anything as the apis are already defined in the proto files

The proto files are not enough, and Dapr also has an HTTP 1.1 API that needs to be taken into consideration here. I agree that if we were to try and formulate a specification out of the Dapr APIs, we'd need more than proto/swagger descriptions.

> Proto文件是不够的，Dapr还有一个 HTTP 1.1的API需要考虑。我同意，如果我们试图从Dapr APIs中制定一个规范，我们需要的不仅仅是 proto/wagger 的描述。

I agree with you that this is a direction to go and it will be interesting if there is a desire to separate out the APIs. It is certainly a direction to keep in mind as Dapr is further adopted. Would like to hear other people's thoughts here.

> 我同意你的观点，如果人们希望将API分离出来，这是一个前进的方向，这将是有趣的。随着Dapr的进一步采用，这肯定是一个值得铭记的方向。希望能在这里听到其他人的想法。

There is an opportunity for Dapr API to mature and become a spec. I believe we will need at least another version of the Dapr API to feel more confident about extracting a spec out of it.

> Dapr API有机会成熟并成为一个规范。我认为，我们至少需要另一个版本的Dapr API，才能更有信心从中提取一个规范。

On the flip side, iterating a spec might lead to some changes in the API.

> 反过来说，迭代一个规范可能会导致API的一些变化。


Thanks for your feedback!

Considering the current adoption rate of http/2, I agree we still need to support the http/1.1 api.
However, I think the api specification for http/2 might be a little different from the one for http/1.1. Because http/2 does have some useful features like streaming, push which can improve the experience for situations like subscribing messages or configuration updates.

I believe we will need at least another version of the Dapr API to feel more confident about extracting a spec out of it.

On the flip side, iterating a spec might lead to some changes in the API.

Right, I also believe it may take some time or even introduce some breaking changes before we finalize the spec. However, I'm wondering is there anything we could do now to help us get closer to the final goal? Please feel free to share your thoughts or ideas, I'll try my best to help.

> 谢谢你的反馈
> 
> 考虑到目前 http/2 的采用率，我同意我们仍然需要支持 http/1.1 的 api。然而，我认为http/2的api规范可能与http/1.1的api规范有些不同。因为http/2确实有一些有用的功能，比如流、push，这可以改善订阅消息或配置更新等情况的体验。
>
> I believe we will need at least another version of the Dapr API to feel more confident about extracting a spec out of it.
> On the flip side, iterating a spec might lead to some changes in the API.
>
> 没错，我也相信在最终确定规范之前，可能需要一些时间，甚至引入一些突破性的变化。然而，我想知道我们现在是否有什么可以帮助我们更接近最终目标？请随时分享你的想法或创意，我会尽力帮助你。

I believe we need to have a schema for the Dapr API first. Right now it is in proto files and hard coded in the http handlers.

> 我相信我们需要先有一个Dapr API的schema。现在它是在proto文件中，硬编码在http处理程序中。
