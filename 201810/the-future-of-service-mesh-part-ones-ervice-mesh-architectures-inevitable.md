---
original: https://blogs.vmware.com/opensource/2018/10/16/service-mesh-architectures-inevitable/
translator: malphi
reviewer: rootsongjc
title: "服务网格的未来 第一部分：服务网格架构势不可挡——并且越来越重要"
description: "本文通过分析阐述了服务网格的未来发展"
categories: "译文"
tags: ["taga"]
date: 2018-09-28

---

## 服务网格的未来 第一部分：服务网格架构势不可挡——并且越来越重要

当[Istio 1.0](https://istio.io/)在几个月前发布时，[TechCrunch](https://techcrunch.com/2018/07/31/the-open-source-istio-service-mesh-for-microservices-hits-version-1-0/)  被称作“可能是目前最重要的开源项目之一”。它并不是完美的(在本系列的第2部分中有详细介绍)，但是这个版本标志着服务网格架构开发的一个重要阶段。

尽管对Istio的发布给予了关注，但是在开源社区，服务网格还是不为人知。因此，在这两篇文章中，我们首先提供一个窗口来了解服务网格的功能，然后在第二部分，询问在不久的将来我们能从它那里得到什么。

One important thing to know about service meshes is that they essentially became inevitable as soon as microservices started to become popular. That’s because, in essence, they operate as platforms for solving the increasingly complex challenge of communicating between these services.

关于服务网格有一件重要的事情需要了解，那就是一旦微服务开始流行起来，它们基本上就不可避免了。这是因为，从本质上讲，它们作为平台运行，可以解决这些服务之间通信的日益复杂的挑战。

Here’s how they work: Say you have one microservice that looks up payment methods in a customer database and another that processes payments. If you want to make sure information doesn’t leak from either of them or that you always connect your customer’s information to the right payment processor, you’ll want to encrypt the traffic between them. A service mesh can take care of that encryption for you without requiring either service to know how to secure that encryption themselves.

它们是这样工作的:假设有一个微服务在客户数据库中查找支付方法，另一个微服务处理支付。如果您希望确保信息不会从它们中泄露出来，或者您总是将客户的信息连接到正确的支付处理器，那么您需要对它们之间的通信进行加密。服务网格可以为您处理加密，而不需要任何一个服务知道如何保护加密本身。

But service meshes do a lot more than just that. Overall, they take care of a wide swathe of core communications features, including:

- ​     Observability – logging and supplying metrics between services
- ​     Discovery – enabling services to be linked together to find other services
- ​     Communication – establishing policy, means and security for communications
- ​     Authentication – establishing access rights to services and communications
- ​     Platform provision – providing control across multiple backends (Azure, AWS, etc.) and orchestrations (Kubernetes, nginx, etc.)

但服务网格的作用远不止于此。总的来说，他们负责广泛的核心通信功能，包括:

-可观察性-在服务之间记录和提供度量
-发现-使服务连接在一起以找到其他服务
-通信-建立通信政策、手段和安全
-认证-建立服务和通信的访问权限
-平台提供-跨多个后端(Azure、AWS等)和编配(Kubernetes、nginx等)提供控制

You can see the appeal for developers—a service mesh takes care of a whole tranche of things they’d rather not have to deal with each time they build a microservice. It’s a boon for sysadmins and deployment teams, too; they don’t have to negotiate with developers to build the features they need into any specific microservice. And customers benefit, in theory at least, because they can deploy their market-specific services much faster.

您可以看到开发人员的吸引力——在每次构建微服务时，服务网会处理掉他们不愿处理的所有事情。这对系统管理员和部署团队来说也是一个福音;他们不必与开发人员协商，将他们需要的特性构建到任何特定的微服务中。至少在理论上，客户会从中受益，因为他们可以更快地部署特定于市场的服务。

Given these advantages, it was basically inevitable that we would get to this point. At first, people created their own communications meshes. Before long, common patterns emerged. Common approaches started to get aggregated and finally took on the form of platform solutions.

考虑到这些优势，我们将不可避免地到达这一点。最初，人们创造了自己的交流网络。不久，共同的模式出现了。公共方法开始得到聚合，最终采用了平台解决方案的形式。

Two years ago, Google open sourced its own service mesh platform as Istio. It wasn’t the first service mesh and isn’t the most mature, but it’s the fastest growing and the debut of 1.0 marks a new stage in the service mesh story.

两年前，谷歌开放了自己的服务网格平台Istio。它不是第一个服务网格，也不是最成熟的，但它是增长最快的，1.0的发布标志着服务网格故事的一个新阶段。

To quote that TechCrunch article again: “If you’re not into service meshes, that’s understandable. Few people are.” But while that may be the case at present, for all the reasons outlined above, we think that’s also very likely to change. It’s why we’re devoting a fair amount of time and energy to contributing to service mesh development here at VMware.

再次引用TechCrunch的那篇文章:“如果你不喜欢服务网络，这是可以理解的。一些人。“但尽管目前情况可能是这样，基于上述所有原因，我们认为这种情况很可能会改变。”这就是为什么我们花了大量的时间和精力在VMware的服务网格开发上。

In part two of this pair of posts, we’ll outline how we are contributing to open source service mesh development at VMware and describe what we see as the major issues these architectures are facing now that they have begun to mature.

在这两篇文章的第2部分中，我们将概述我们如何在VMware的开源服务网格开发中做出贡献，并描述我们认为这些体系结构在开始成熟后所面临的主要问题。

Stay tuned to the Open Source Blog for part two of our service mesh blog series and follow us on Twitter (@vmwopensource).*

请继续关注我们的服务mesh博客系列的第二部分的开源博客，并在Twitter上关注我们(@vmwopensource)。