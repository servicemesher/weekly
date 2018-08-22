
> 原文链接：http://blog.christianposta.com/microservices/application-safety-and-correctness-cannot-be-offloaded-to-istio-or-any-service-mesh/
>
> 发布时间：2018-08-10
> 
> 作者：Christian Posta
> 
> 译者：陈冬
> 
> 审校：


# 应用程序的安全性和正确性不能卸载到 Istio 或任意的服务网格中

我最近在讨论集成服务的演进以及服务网格的使用，特别是关于 Istio 。自从2017年1月我听说了 Istio 以来，我一直很兴奋，事实上我是为这种新技术感到兴奋，它可以帮助组织构建微服务以及原生云架构成为可能。也许你可以说，因为我已经写了很多关于它的文章（请关注 [@christianposta](https://twitter.com/christianposta)的动态)：

* [The Hardest Part of Microservices: Calling Your Services](http://blog.christianposta.com/microservices/the-hardest-part-of-microservices-calling-your-services/)
* [Microservices Patterns With Envoy Sidecar Proxy: The series](http://blog.christianposta.com/microservices/00-microservices-patterns-with-envoy-proxy-series/)
* [Application Network Functions With ESBs, API Management, and Now.. Service Mesh?](http://blog.christianposta.com/microservices/application-network-functions-with-esbs-api-management-and-now-service-mesh/)
* [Comparing Envoy and Istio Circuit Breaking With Netflix OSS Hystrix](http://blog.christianposta.com/microservices/comparing-envoy-and-istio-circuit-breaking-with-netflix-hystrix/)
* [Traffic Shadowing With Istio: Reducing the Risk of Code Release](http://blog.christianposta.com/microservices/traffic-shadowing-with-istio-reduce-the-risk-of-code-release/)
* [Advanced Traffic-shadowing Patterns for Microservices With Istio Service Mesh](http://blog.christianposta.com/microservices/advanced-traffic-shadowing-patterns-for-microservices-with-istio-service-mesh/)
* [How a Service Mesh Can Help With Microservices Security](http://blog.christianposta.com/how-a-service-mesh-can-help-with-microservices-security/)

Istio 建立在容器和 Kubernetes 的一些目标之上：提供有价值的分布式系统模式作为语言无关的习惯用法。例如：Kubernetes 通过执行启动/停止、健康检查、缩放/自动缩放等来管理容器，而不管容器中实际运行的是什么。类似的， Istio 可以通过在应用程序容器之外透明地解决可靠性、安全性、策略和通信量方面的挑战。

随着 [Istio 1.0](https://istio.io/blog/2018/announcing-1.0/) 版本在2018年7月31日的发布，我们看到 Istio 的使用和采纳有了很大的增加。我看到的一个问题是“如果 Istio 为我提供了可靠性，那么我还需要爱应用程序中担心它吗？”

我看到的答案是：绝对要

[几乎就在前一年，我写了一片文章](http://blog.christianposta.com/microservices/application-network-functions-with-esbs-api-management-and-now-service-mesh/)，其中包含了这一区别，但并不是足够有力；这篇文章是我试图帮助纠正这一点，[并建立在前面提到的谈话的基础上](https://www.slideshare.net/ceposta/evolution-of-integration-and-microservices-patterns-with-service-mesh-107786281)。

因此，设置一些上下文：Istio 提供了应用程序网络的“可靠性”能力，例如：
* 自动重试（automatic retry）
* 重试 定额/预算（retry quota/budget）
* 连接超时（connection timeout）
* 请求超时（request timeout）
* 客户端负载均衡（client-side load balancing）
* 电路断开（circuit breaking）
* 隔离（bulkheading）

在处理分布式系统时，这些功能是必不可少的。网络并不是可靠的，并且破坏了我们在一个整体中所拥有的很多好的安全假设/抽象。我们要么迫切的解决这些问题，要么遭受系统范围内不可预测的停机。

## 退一步说

这里更大的问题实际上是让应用程序相互通信来解决一些业务的问题。这就是为什么我们编写软件的原因，最终用来提供某种商业价值。同时该软件也使用一些商业领域的结构，例如：“客户”、“购物车”、“账户”等。从领域驱动的设计来看，每个服务可能在理解这些概念上都略微有不同。

这里有一些不太明确的概念和更大的业务约束（例如：客户可以由名称和电子邮件来确认唯一性，或者客户只能拥有一种类型的支票账户等），以及不可靠的网络和整个不可预知的基础设施（假定可以构建这样的服务，可以或者失败）使构建正确事情是非常困难的。

端到端的正确性和安全性

然而，事实上是，在构建正确和安全的应用程序方面，这样的责任归属到了应用程序上（以及所有它支持的人）。我们可以尝试将更低级别的可靠性构建到系统的性能或优化的组件中，但总的责任还是在应用程序中。1984年，Saltzer ， Reed 和 Clark 在“系统设计中的端到端论证”中提到了这一原则。具体地说：

> 只有在通信系统端点的应用程序的知识和帮助下，才能完全正确的实现所讨论的功能。

在这里，“功能”是应用程序的需求之一，比如“预订”或“像购物车中添加商品”。这种功能不能概括为通信系统或其组建/基础设置（这里的通信系统指的是网络、中间件和任何为应用程序提供基础设施的工作）：

> 因此，提供被质疑的功能作为通信系统本身的特质是不可能的。

然而，我们可以做一些事情以保证通信系统的部分可靠，这样有助于实现更高层次的应用程序的需求。我们做这些事情后可以优化部分区域，以至于不至于太多的担心这样的问题，但这也不是应用程序可以完全忽略的事情：

> 有时，通信系统提供的功能不完整的版本可能作为性能增强有用

例如：在 Saltzer 的论文中，他们使用从应用程序 A 传输文件到应用程序 B 的示例：

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fuibtn8kvfj31fc0v2ju4.jpg)

我们需要做什么（安全）来保证文件被正确的传送到？在图中的任何一点都有能出现错误：1）存储机制可能有失败的区域/移位的比特/损坏，所以当应用程序 A 读取一个文件时，读取的是一个错误的文件；2）应用程序在读取文件到内存中或者发送文件时存在 bug ；3）网络可能混淆字节的顺序，文件部分重复等。我们可以进行优化，例如使用更可靠的传输，如 TCP 协议或消息队列，但是 TCP