# The Circonus Istio Mixer Adapter

> 原文地址：<https://www.circonus.com/2017/12/circonus-istio-mixer-adapter/>
>
> 作者：Fred Moyer
>
> 译者：[陈冬](https://github.com/shaobai)


在 Circonus，我们有悠久的开源软件参与的传统。因此，当我们看到 Istio 提供了一个精心设计接口，通过适配器连接 syndicate 服务遥测，我们知道一个 Circonus 适配器将是一个自然的契合。Istio 已经被设计成提供高性能、高可扩展的应用控制平面，并且 Circonus 也被设计为具有性能和可扩展行的核型原则。

今天我们很高兴的宣布 [Istio 服务网格](https://istio.io/) 的 Circonus 适配器的可用性。这篇博客文章将介绍这个适配器的开发，并向您展示如何快速启动并运行它。我们知道你会对此非常有兴趣，因为 Kubernetes 和 Istio 提供你能力扩展到 Circonus 设计的水平，高于其他遥测解决方案。

如果你不知道什么是服务网格，你并不孤单，但希望是你已经使用很多年了。互联网的路由基础设施就是一个服务网格；它有利于 TCP 重传、访问控制、动态路由、流量规划等。占主导地位但web整体性web应用正在为微服务组成的应用让路。Istio 通过一个  [sidecar proxy](https://www.envoyproxy.io/docs/envoy/latest/) 提供基于容器的分布式应用程序的控制平面功能。它为服务的操作人员提供了丰富的功能来控制 [Kubernetes](https://kubernetes.io/) 编排的服务集合，而不需要服务本身来实现任何控制平面的功能集合。

 Istio 混合器 [Mixer](https://istio.io/docs/concepts/policies-and-telemetry/overview/) 提供了一个 [适配器](https://istio.io/blog/2017/adapter-model/) 模型，它允许我们创建用于，它允许我们通过创建用于外部基础设施后端接口的混合器的 [处理器](https://istio.io/docs/concepts/policies-and-telemetry/config/#handlers) 来开发适配器。混合器还提供了一组模版，每个模板都为适配器提供了不同的元数据集。在例如 Circonus 适配器之类的度量适配器，该元数据集包括诸如*请求持续时间*（*request duration*）、*请求计数*（*request count*）、*请求有效负载大小*（*request payload size*）等度量。要激活 Istio 启用的 Kubernetes 集群中的 Circonus 适配器，只需要使用 istioctl 命令将 Circonus  [操作员配置](https://github.com/istio/istio/blob/master/mixer/adapter/circonus/operatorconfig/config.yaml)(`地址有误，待修正`) 注入到 k8s 集群中，度量（metrics）将开始流动。

 以下是一个关于混合器如何与这些外部后端服务交互的架构视图：
![]（https://ws4.sinaimg.cn/large/006tNc79gy1ft2kovaczfj319u0zftc9.jpg)
