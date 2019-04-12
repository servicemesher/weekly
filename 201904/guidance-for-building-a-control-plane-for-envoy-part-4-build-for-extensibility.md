---
author: "Christian Posta"
translator: "haiker2011"
reviewer: [""]
original: "https://medium.com/solo-io/guidance-for-building-a-control-plane-for-envoy-part-4-build-for-extensibility-40f8ac8e48e"
title: "为 Envoy 构建控制面指南第4部分：构建的可扩展性"
description: "本文介绍如何为 Envoy 构建控制面指南的第4部分：构建的可扩展性。"
categories: "translation"
tags: ["Envoy", "Control Plane", "Gloo", "Service Mesh"]
originalPublishDate: 2019-04-12
publishDate: 2019-04-12
---

这是探索为 Envoy 代理构建控制面系列文章的第4部分。

在本系列博客中，我们将关注以下领域:

* [采用一种机制来动态更新 Envoy 的路由、服务发现和其他配置](https://medium.com/solo-io/guidance-for-building-a-control-plane-to-manage-envoy-proxy-at-the-edge-as-a-gateway-or-in-a-mesh-badb6c36a2af)

* [确定控制面由哪些组件组成，包括支持存储、服务发现 api、安全组件等](https://medium.com/solo-io/guidance-for-building-a-control-plane-for-envoy-proxy-part-2-identify-components-2d0731b0d8a4)

* [建立最适合您的使用场景和组织架构的特定于域的配置对象和 api](./Guidance-for-Building-a-Control-Plane-for-Envoy-Part-3-Domain-Specific-Configuration.md)

* 考虑如何最好地使您的控制面可插在您需要它的地方(本博客)

* 部署各种控制面组件的选项

* 通过控制面的测试工具来思考

在[上一篇文章]()中，我们探讨了为您的控制平面构建一个特定于领域的API，该API最适合您的组织和工作流首选项/约束。

## 构建可插拔的控制平面引擎

Envoy是一个非常强大的软件，每天都有[新的用例和新的贡献被提交给社区](https://github.com/envoyproxy/envoy/pull/4950)。尽管Envoy的核心非常稳定，但它建立在[可插拔的过滤器架构](https://github.com/envoyproxy/envoy-filter-example)之上，因此人们可以为不同的L7协议编写新的编解码器或添加新的功能。目前，Envoy过滤器是用C++编写的，可以选择使用[Lua](https://www.envoyproxy.io/docs/envoy/latest/configuration/http_filters/lua_filter)扩展Envoy，但是也有[一些讨论支持Web Assembly实现](https://github.com/envoyproxy/envoy/issues/4272)可扩展性。同样值得注意的是，[Cilium](https://cilium.io/)的伟大人士正在围绕一个[基于Go的Envoy可扩展机制](https://cilium.io/blog/2018/10/23/cilium-13-envoy-go/)开展工作。除了快速移动的Envoy社区和配置这些新功能的需要之外，还需要包括新的特定于领域的对象模型，以支持希望利用Envoy的新平台。在本节中，我们将探索沿着这两个维度扩展Envoy控制平面。

通过编写C++过滤器，扩展Envoy非常简单。我们在[Gloo项目](https://github.com/solo-io/envoy-gloo)上创建的特使过滤器包括：

* [Squash](https://github.com/solo-io/squash)调试器
(https://github.com/envoyproxy/envoy/tree/master/api/envoy/config/filter/http/squash)

* Caching(目前为封闭源码;应该在不久的将来开放源代码)

* Request/Response 传输 (https://github.com/solo-io/envoy-gloo/tree/master/source/extensions/filters/http/transformation)

* AWS lambda (https://github.com/solo-io/envoy-gloo/tree/master/source/extensions/filters/http/aws_lambda)

* NATS streaming (https://github.com/solo-io/envoy-nats-streaming, https://github.com/solo-io/envoy-gloo/tree/master/source/extensions/filters/http/nats/streaming)

* Google Cloud Functions (https://github.com/solo-io/envoy-google-function)

* Azure function (https://github.com/solo-io/envoy-azure-functions)

![](https://ws1.sinaimg.cn/large/006gLaqLgy1g202enyhmsg30q10iajsu.gif)

在上面的图示中，我们可以看到请求是如何通过Envoy的，以及如何通过一些过滤器的，这些过滤器具有应用于请求和响应的特定任务。你可以在[Solo.io](https://www.solo.io/)首席执行官/创始人[Idit Levine](https://medium.com/@idit.levine_92620)和Solo.io首席架构师[Yuval Kohavi](https://medium.com/@yuval.kohavi)写的一篇博客文章中读到更多关于[Envoy的功能和我们为构建Gloo的控制平面所做的权衡](https://medium.com/solo-io/building-a-control-plane-for-envoy-7524ceb09876)。

因为Envoy功能非常多，而且一直在添加新特性，所以值得花一些时间来考虑是否要将控制平面构建为可扩展的，以便能够使用这些新特性。在Gloo项目中，我们选择在以下几个层次上进行：

* 在核心Gloo配置对象的基础上构建更自定义的特定于域的配置对象

* 控制平面插件以增强控制平面的现有行为

* 创建工具来加速前面两点

让我们来看看每一个层次，以及它们如何构成可扩展和灵活的控制平面。

## 核心API对象，构建时考虑灵活性

## 扩展特定于域的配置层

## 控制平面插件以增强控制平面的现有行为

## 利用工具加快前面两个要点

## 小结

控制平面可以简单到您需要的程度，也可以复杂到您需要的程度。Gloo团队建议将重点放在控制平面的简单核心上，然后通过可组合性通过插件和微服务控制器扩展它。Gloo的体系结构是这样构建的，它使[Gloo团队](https://github.com/solo-io/gloo/graphs/contributors)能够快速添加任何新特性，以支持任何平台、配置、过滤器，以及更多的新特性。这就是为什么，尽管Gloo是非常kubernets原生的，但它是为在任何云上的任何平台上运行而构建的。核心控制平面的设计允许这样做。

在本系列的下一篇文章中，我们将讨论部署控制平面组件的优缺点，包括可伸缩性、容错、独立性和安全性。请[继续关注](https://twitter.com/soloio_inc)!