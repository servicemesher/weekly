---
original: http://wei-meilin.blogspot.com/2019/03/my2cents-eight-things-leads-to.html
author: "CHRISTINA の J老闆"
translator: "malphi"
reviewer: ["microservice"]
title: "我的两分钱 - 八件事导致开发灾难性的云原生微服务系统"
description: "本文介绍了8种开发云原生微服务系统中出现的错误"
categories: "translation"
tags: ["microservice"]
originalPublishDate: 2019-03-19
publishDate: 2019-04-16
---

# 我的两分钱 - 八件事导致开发灾难性的云原生微服务系统

大部分标注“我的两分钱”的文章只是我的想法。你只需要快速愉快的阅读，不用太深入，但值得做笔记：）

## 1. 设置错误的领域边界

这是一种工作保证策略，它让参与项目的每个人在开发和测试中无休止地循环，而无法将服务投入生产环境！首先，一切都从简单开始，逐渐发现有越来越多的功能、业务逻辑被添加到微服务中，最后甚至不得不重新命名整个该死的东西。

![1](https://ws1.sinaimg.cn/large/006tNc79ly1g23ajtc2kej305k057glt.jpg)

<u>临床症状和副作用</u>

- 不断增长的微服务变得过于臃肿，或者域中的每个微服务都调用你的微服务。（有时核心微服务具有相同的行为，但你不应该在单个域中看到许多此类服务）。这违反了简单、可维护和敏捷的微服务原则。
- 到处都是重复的微服务/代码。你可以找到一些重复的代码或微服务，它们被复制和部署到其他域中。

![2](https://ws3.sinaimg.cn/large/006tNc79ly1g23ajj6lw8j305k03swei.jpg)

如果您陷入了无休止的实现和测试地狱，退一步看看如何分隔域。你是acc标识性的设置来自其他域的上下文或混合不同概念到一个里面？也许回到设计阶段考虑边界是值得的。为了避免到处重复，请确保有适当的文档，比如在域之间可用的使用开放API标准的文档。

## 2. 混合微服务的职责

想吃意面的话这是一个不错的选择。在意面上放些肉丸子让它更美味？在服务中混合一些有状态的进程将会给您带来很多!

j

- ![3](https://ws2.sinaimg.cn/large/006tNc79ly1g24bk7332yj308w07kmy0.jpg)

-   组合微服务
-   功能性的核心业务
-   无状态整合
-   有状态的进程

<u>临床症状和副作用</u>

- 客户端调用变得复杂，如果不提供特定服务的组合视图，客户端可能需要组合并在其中处理某种核心业务。
- 服务之间的关系就好像意大利面条的关系，而且很容易无法追踪数据流和业务逻辑是如何在微服务周围传播的。 
- 太多的耦合，无状态和有状态依赖于存储解决方案，向下扩展也可能有潜在问题。

我写了很多关于这一主题的博客，主要观点都是担心拆分，确保你的云原生微服务是干净、精练和敏捷的，这意味着你需要确保业务组合和实际业务功能需要单独分开到不同的实例和独立部署，以便更好的结构化和模块化。为了可伸缩性和灵活性，确保微服务的组合/编排是无状态的。用专门的存储解决方案在其他实例上保留长时间运行的进程，以保持状态。



## 3. 和外界因素/需求太多的耦合





Being patient is always good, if you want to practice your ability to be extra patient while being calm, by coupling deeply with legacy system will train you to become the patient master, and become better at code management as you need juggle between agile demands and slow legacy update cycles.

Symptoms and side effects

- Constant updates to core business to the core business system outside of better control. Having external system decide and make change request and short reaction time could cause issues. 
- Once contract are establish between parties, will need extra care for version updates. 

![img](https://2.bp.blogspot.com/-oGhZ34E1fd4/XJD4_39hLMI/AAAAAAAAF2o/z3Nc1VgeHiI6AgERXs3GIEevv_CqNLerwCLcBGAs/s320/imageedit_52_6603423796.png)

The deployment cycle varies from system to system, but brown field application will have a longer application lifecycle compare to more agile and dynamic cloud native application. To move things faster, always consider to have a shield protection between your brown/green field application. So it accommodates to the s

lower moving parts as well as help green field cycle to continue on it's agile development. In your core domain microservices, avoid add anything that is dependent or specific for external consumers, as it might leads to too much customization that breaks the definite context boundary. Make sure you have other vendor/external system facing module that does the customization and hide the presentation complexity from internal domain models.

\4. 

Duplicating and replicate non-biz related code in microservices

As a developer, it's nice to be depended on, you feel the satisfaction of being needed. There is a reason why there are 24 hours each day, so you get to work 20 hours per day and you are so important when ever there is a deployment, you needed to be there. 

Symptoms and side effects

[![img](https://3.bp.blogspot.com/-iuEUUJ0k9OI/XJD4_8xgIPI/AAAAAAAAF2s/mOcswnwz3SwUQj2jWDfVKtr6YAVgXcEtACLcBGAs/s320/imageedit_54_8534297864.png)](https://3.bp.blogspot.com/-iuEUUJ0k9OI/XJD4_8xgIPI/AAAAAAAAF2s/mOcswnwz3SwUQj2jWDfVKtr6YAVgXcEtACLcBGAs/s1600/imageedit_54_8534297864.png)

- None centralized control, all retry, reroute, versioning and deployment strategy are store in each individual instance.
- Responsibility not clear between DEV/DEVOPS, again, the networking strategy is better for party that has better access to monitoring stats of how the environment is doing. 

Setting retry and routing policy can allows application to be more robust to failure. To have a more centralized view of how these policies are handled instead of spreading them all over individual microservices. Release the burden of developer handling everything in the application and better management and monitoring.

\5. 

Service in mesh, connecting all with APIs

Making API calls are super easy, and service mesh is what everyone is doing today. Every technology has an library to handle REST/JSON. Let's connect all components in the system with APIs! 

Symptoms and side effects

- Slow to react, since request and response are how most people uses it. And the wait could take longer and more locks for longer process. That becomes the performance bottleneck. 
- Not utilizing cloud infrastructure. 

When microservice first came out, people wants to get away from ESB therefore many moved away from messaging broker and use API as the only connection method between services, as the Open API standard allows you to build a nice catalog among what is available. But event driven is there for a reason, to achieve true scalability, better decoupling, you need to stop making sticky dependency between calls. And event driven allows you to populate events to related parties, and because it's asynchronies, no resource are wasted when waiting for the reply.

\6. 

Un-sorted events

![img](https://3.bp.blogspot.com/-wV8qmDwaGjM/XJD6Ou07sTI/AAAAAAAAF3I/_7b1Uhwdnts7mgV3cbNHiq-1CHfy4KZsACLcBGAs/s320/imageedit_58_5805145960.png)

Blasting the system with events, notify every corner to any events, will get your system super reactive! By the way, why not stream everything and store all the event! For better traceability!

Symptoms and side effects

- Events everywhere, you realized you have to listen to insane amount of events to react up on it. 
- Un-needed code to filter out events.
- Confusion between status change or action should taken
- Don't know what is better to preserve or lose the event messages

There are various types of events, if not carefully where and how the events are distributed, it's very likely to end up with unwanted events floating around consuming unnecessary resources. Focusing on some of the common ones like events containing data, status and commands. You want to minimize the number of data messages, as it take longer to process, more work to store. Striping down, filter the data could be more efficient. Handling retries of status events and rolling back command events should also be part of the event strategies.

\7. 

Data Silos

One mircroservice one datasource, that is what experts says. 

Symptoms and side effects

- Data not consistent among the datasources, or slow. 
- Un-needed microservices created just for making sure the data are consistent.
- Events everywhere to update status.

![img](https://1.bp.blogspot.com/-DOV4sPgmz4c/XJD6Orec7VI/AAAAAAAAF3M/lCFtFLbW9DURh-9eYe4KGy7BzKluQTqxACLcBGAs/s320/imageedit_56_4470457635.png)

When you have isolated datasources there are risk of creating data silos, and because of the nature in microservice being distribute and more complex datastore scenario, it has become more difficult to mange the data consistency. You can start looking into how to capture the data change with many existing solutions such as streaming event changes as it happens during process or listen to changes from the main status store.

8.

 Limited automation

Manual is much more flexible, beside that was how it was done before with the JavaEE application on the servers. Automation, it takes too much time to setup, Lets deal with it when we have time?  

Symptoms and side effects

- Slow updates
- Long painful steps to productions

The first thing that needs to be in the planning is definitely automation. How to apply automated CI/CD pipelines and the deployment strategy, this how cloud native microservices system achieve the agility. I've done a couple of CI/CD automation with Jenkins in the past. 

See Github Link

.

Alright, these are just my two cents, let me know if you think there are any other things that could lead to terrible cloud native microservices development? Love to know!