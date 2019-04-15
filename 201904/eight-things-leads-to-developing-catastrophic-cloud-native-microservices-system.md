---
original: http://wei-meilin.blogspot.com/2019/03/my2cents-eight-things-leads-to.html
author: "CHRISTINA の J老闆"
translator: "malphi"
reviewer: ["microservice"]
title: "我的两美分 - 八件事导致开发灾难性的云原生微服务系统"
description: "本文介绍了8种开发云原生微服务系统中出现的错误"
categories: "translation"
tags: ["microservice"]
originalPublishDate: 2019-03-19
publishDate: 2019-04-16
---

More of my two cents, just my thoughts. A quick fun read, not too deep, but worth noting :).

\1. 

Setting the domain boundary wrong





This is a job guarantee tactics, it's endless looping in development and testing for everyone involved in the project without making the service to production! First everything starts simple and gradually find more and more functions, business logic gets added into the microservice, at the end, one even have to rename the whole damn thing. 

![img](https://3.bp.blogspot.com/-cMJw865hXvs/XJD4_BOc3MI/AAAAAAAAF2k/l2pxr-xYjGcNQp0XVJ3bYwfmhmcR5HnCwCLcBGAs/s200/imageedit_44_4536471448.png)

Symptoms and side effects 

- A growing microservices becomes too fat, or every single microservices in the domain calls your microservice. (Sometime core microservice has the same behavior, but you should not see many of this type of services in a single domain.) This violates the microservices principle of easy, maintainable and agile.
- Duplication of microservices/code everywhere. Where you find bits and pieces of duplicate code or microservices being copied and deployed into other domains. 

![img](https://2.bp.blogspot.com/-RyZm0TiTguw/XJD4_IEPI9I/AAAAAAAAF2c/8rz8gE1DIPoMr8BhxJppPLb79P_CjYGhACLcBGAs/s200/imageedit_48_9341058924.png)

If you get into the endless implementation and testing hell, take a step back and look at how you separate the domains. Did you acc

identally place a context from other domains or mixing different concepts int

o one? Maybe it's worthwhile to go back to the design phase and think about the boundaries. To avoid duplication everywhere, make sure there are proper documented catalogs such as API portals using the Open API Standard Doc available between domains.

\2. 

Mixing responsibilities of microservices 

In the mood for some Spaghetti, this is the way to go, and want some meatball to go on the Spaghetti to make it tastier? Mixing some stateful processes in there will get you plenty! 

Mixing

- ![img](https://3.bp.blogspot.com/-DMPAmCWjHDg/XJD4_MjWflI/AAAAAAAAF2g/q_CN82QbZZoaP6raYZXxDETb7svYd6bgACLcBGAs/s320/imageedit_50_9140009164.png)

-   Composing microservices
-   Functional core business
-   Stateless integration
-   Stateful process

Symptoms and side effects 

- Client calls becomes complex, if you don't provide a composed view of particular service, the client may require to compose and possibly handle some sort of core business login in it. 
- Spaghetti relations between services, and it's really easy to lose track of how the flow of data and business logic spread around microservices.  
- Too much coupling, having stateless and stateful is depended on the storage solution and scaling down could also have potential problems.

I've done a bunch of blogs on this topic, the main concept is always the 

separation of concerns

, making sure there your cloud native microservices are clean, lean and agile meaning you need to make sure the business composition and the actual business functions needs to be separated into different instance and deployed individually for better structured and more modularized. Making sure the composition/orchestrations of microservices are stateless for scalability and flexibility. Set aside other long running process on other instance with dedicated proper storage solution to keep the states.

\3. 

Too much coupling with external factors/demands





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