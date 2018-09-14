# Istio 101：“服务网格的未来是与Knative和Apache Whisk等技术共生的”

> 原文链接：https://jaxenter.com/istio-service-mesh-interview-harrington-148638.html
> 作者：[Gabriela Motroc](https://jaxenter.com/istio-service-mesh-interview-harrington-148638.html#authors-block)
> 译者：殷龙飞

Istio正在引起很多关注，特别是现在1.0就在这里。 但它是否具备成为Kubernetes事实上的服务网络所需要的东西？ 如果你问Red Hat的Istio产品经理Brian'Redbeard'Harrington，答案是肯定的。 “有了Istio，部署很简单，与Kubernetes的整合是一流的。 感觉好像应该一直存在。“

Istio [1.0](https://jaxenter.com/istio-1-0-arrived-core-features-ready-production-use-147459.html) 本月初到达; 所有 [核心功能](https://istio.io/about/feature-stages/)  现在都可以用于生产。

如果您已经熟悉0.8中提供的功能，那么您应该知道1.0中提供的新功能列表并不长; 该团队选择专注于修复错误并提高性能。 如果您希望看到Istio 1.0中引入的所有更改，我邀请您阅读 [发行说明](https://istio.io/about/notes/1.0/) 。

*我们与Red Hat的Istio产品经理Brian'Redbeard'Harrington讨论了他最喜欢的功能，Istio的未来以及它是否具备成为Kubernetes事实上的服务网络所需的功能。*

## Istio：改变游戏规则？

**JAXenter：Istio可能相对较新，但这种用于连接，管理和保护微服务的工具正在获得动力。增长背后的原因是什么？**

**Brian'Redbeard'Harrington：** 最大的原因是范式的转变。 在 [Netflix的OSS](https://netflix.github.io/) （开放源代码软件套件）带来了很多强大的功能，个人开发企业Java应用程序，但它要求你为了实现整个套件的利益整合库的聚宝盆。 Istio令人兴奋，因为它为用户提供了A / B测试，断路，服务授权等功能，同时最大限度地减少了代码更改。

**JAXenter：Google最近宣布的[云服务平台](https://jaxenter.com/google-cloud-interesting-announcements-147230.html)以Istio（和Kubernetes）为核心。这对Istio的未来意味着什么？**

**Brian'Redbeard'Harrington：** 这表明该领域的老牌企业已经认识到了一项卓越的技术，并且明白早期合作将为客户带来更多成功。 反过来，如果客户成功，他们将增加该供应商提供的解决方案的采用。

**JAXenter：Istio能否成为Kubernetes事实上的服务网络？**

**Brian'Redbeard'Harrington：** 我绝对相信它。 其他解决方案通常是在操作组件，这些组件不是以云原生主体为基础构建的，因此可能总是感觉有点笨拙。 使用Istio，部署非常简单，与Kubernetes的集成是一流的。 感觉好像应该一直存在。

**JAXenter：你最喜欢的功能是什么包含在Istio 1.0中？**

**Brian'Redbeard'Harrington：** 仍然让我的袜子脱落的功能是能够控制分流量的路由。 当我过去运行服务时，这个组件总是需要昂贵的专用负载平衡硬件的组合以及对我的应用程序的修改（并且经常将它们抛弃并重新开始以使其运行良好）。

在Istio中，将10％的流量分配到不同版本的服务并将这些连接路由到该版本的服务是微不足道的。 围绕该功能的易用性改变了我的游戏规则。

**还请参见： [Istio 1.0已经到货：所有核心功能都可以用于生产](https://jaxenter.com/istio-1-0-arrived-core-features-ready-production-use-147459.html)**

**JAXenter：模块化是Istio未来的一部分吗？**

**Brian'Redbeard'Harrington：** 模块化是今天Istio现实的一部分。 Istio规定了某些需要满足的接口，然后允许用户使用他们最熟悉的软件来满足这些接口。 这在“Nginmesh”项目中最为明显，其中Envoy（Istio的代理组件）被Nginx取代。

其他用户同样用Linkerd取代了Envoy。

**JAXenter：Istio最重要的好处是什么？**

**Brian'Redbeard'Harrington：** Istio真正闪耀的一个主要领域是它专注于应用程序的安全性。 设置双向TLS的功能可自动解锁其他优势，例如服务授权以及服务之间的加密。 Istio还具有与其他 [SPIFFE](https://spiffe.io/) （适用于所有人的安全生产身份框架）兼容系统 集成的能力， 这将有助于推动未来采用更高度安全的应用程序。

随着时间的推移，我希望看到安全故事进一步扩展，包括类似于Google的 [身份识别代理的功能](https://cloud.google.com/iap/) 。 关于这一点的好处是，已经通过对JSON Web令牌的支持和对OpenID Connect的支持奠定了一些基础。

**还请参见： [Google Cloud Next '18：云开发人员所希望的一切](https://jaxenter.com/google-cloud-interesting-announcements-147230.html)**

**JAXenter：Istio对Linkerd没有什么帮助？**

**Brian'Redbeard'Harrington：** Istio拥有一个蓬勃发展的社区，正以惊人的速度增长。 顺便提一下，Istio已经存在了大约 [21个月](https://github.com/istio/istio/commit/0216e811e9da88b867742710f7d166cef2eabfbc) ， 在GitHub上 有超过200个个人贡献者和一个非常活跃的“ [脉搏](https://github.com/istio/istio/pulse) ”（即使你只看其核心项目而忽略像Fortio这样的子项目）。

Linkerd已经存在了近 [31个月](https://github.com/linkerd/linkerd/tree/37e38f2a892d9354eea7305135aa6370612b02f2) 。 即使你结合 [Linkerd v1](https://github.com/linkerd/linkerd/pulse) 和 [Linkerd v2](https://github.com/linkerd/linkerd2/pulse/) 的“脉冲” ，它们仍然比Istio的社区活动苍白。

**JAXenter：服务网格的未来是什么样的？**

**Brian'Redbeard'Harrington：** 我相信服务网格的未来与无服务器计算有关。 我们正在融合开发人员成功地将代码库分解为原子组件的状态。

这种趋势甚至反映在围绕Istio模块化的问题上。 我觉得服务网格的未来是与Knative和Apache Whisk等技术共生的，它使开发人员能够重新采用“做一件事，做得好”的“UNIX思想”，以建立应用的未来。

**谢谢！**