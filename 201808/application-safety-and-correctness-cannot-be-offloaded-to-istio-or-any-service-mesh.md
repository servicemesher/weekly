---
原文链接：http://blog.christianposta.com/microservices/application-safety-and-correctness-cannot-be-offloaded-to-istio-or-any-service-mesh/
发布时间：2018-08-10
作者：Christian Posta
译者：陈冬
审校：
---

## 应用程序的安全性和正确性不能卸载到 Istio 或任意的服务网格中

我最近在讨论集成服务的演进以及服务网格的使用，特别是关于 Istio 。自从2017年1月我听说了 Istio 以来，我一直很兴奋，事实上我是为这种新技术感到兴奋，它可以帮助组织构建微服务以及原生云架构成为可能。也许你可以说，因为我已经写了很多关于它的文章（请关注 [@christianposta](https://twitter.com/christianposta)的动态)：

* [The Hardest Part of Microservices: Calling Your Services](http://blog.christianposta.com/microservices/the-hardest-part-of-microservices-calling-your-services/)
* [Microservices Patterns With Envoy Sidecar Proxy: The series](http://blog.christianposta.com/microservices/00-microservices-patterns-with-envoy-proxy-series/)
* [Application Network Functions With ESBs, API Management, and Now.. Service Mesh?](http://blog.christianposta.com/microservices/application-network-functions-with-esbs-api-management-and-now-service-mesh/)
* [Comparing Envoy and Istio Circuit Breaking With Netflix OSS Hystrix](http://blog.christianposta.com/microservices/comparing-envoy-and-istio-circuit-breaking-with-netflix-hystrix/)
* [Traffic Shadowing With Istio: Reducing the Risk of Code Release](http://blog.christianposta.com/microservices/traffic-shadowing-with-istio-reduce-the-risk-of-code-release/)
* [Advanced Traffic-shadowing Patterns for Microservices With Istio Service Mesh](http://blog.christianposta.com/microservices/advanced-traffic-shadowing-patterns-for-microservices-with-istio-service-mesh/)
* [How a Service Mesh Can Help With Microservices Security](http://blog.christianposta.com/how-a-service-mesh-can-help-with-microservices-security/)

Istio 建立在容器和 Kubernetes 的一些目标之上：提供有价值的分布式系统模式作为语言无关的习惯用法。例如：Kubernetes 通过执行启动/停止、健康检查、缩放/自动缩放等来管理容器，而不管容器中实际运行的是什么。类似的， Istio 可以通过在应用程序容器之外透明地解决可靠性、安全性、策略和通信量方面的挑战。

随着 Istio 1.0 版本在2018年7月31日的发布，我们看到 Istio 的使用和采纳有了很大的增加。我看到的一个问题是“如果 Istio 为我提供了可靠性，那么我还需要爱应用程序中担心它吗？”

