# 我需要服务网格吗？

“服务网格”是一个热点话题。似乎去年每一个与容器相关的大会都包含了一个“服务网格”议题，世界各地有影响力的业内人士都在谈论这项革命性的技术带来的好处。

然而，截至2019年初，服务网格技术仍不成熟。主要的实现产品Istio还没有准备好进行广泛的企业级部署，只有少数成功的生产环境在运行。也存在其他的服务网格产品，但并没有得到业界专家所说的广泛关注。

我们如何协调这种不匹配呢？一方面，我们听到“你需要一个服务网格”的声音，而另一方面，企业和公司多年来一直在没有服务网格的容器平台上成功地运行着它们的应用。

## 开始使用 Kubernetes

*服务网格是你旅途中的一个里程碑，但它不是起点。*

在容器应用的生产环境部署中，Kubernetes已经被证明是一个可以胜任的平台。它提供了一个丰富的网络层，提供了[服务发现](https://kubernetes.io/docs/concepts/services-networking/service/#discovering-services), [负载均衡](https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies), [健康检查](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) 和[访问控制](https://kubernetes.io/docs/concepts/services-networking/network-policies/) 的能力以支持复杂的分布式系统。

这些功能对于简单的和易于理解的应用程序来说已经足够了， [遗留的应用已经被容器化](https://www.docker.com/solutions/MTA). 它们允许你满怀信心地部署应用，根据需要扩容，避免意外故障，并实现简单的访问控制。

![1](https://ws1.sinaimg.cn/large/006tKfTcly1g1byouk0a6j30sg0da3zi.jpg)① Kubernetes 提供了带有服务发现和负载均衡的4层网络。② NGINX入口控制器负责把外部连接负载均衡到运行在Kubernetes集群的服务。

Kubernetes在它的API中提供了一个入口（Ingress）资源对象。 这一对象定义了如何选择可以被集群外部访问的服务，一个入口控制器实现了那些策略。 NGINX作为大多数实现中负载均衡的选择，我们为开源的NGINX和NINGX Plus都提供了[高性能、可支持的、生成环境的实现](https://www.nginx.com/products/nginx/kubernetes-ingress-controller/)。

对很多线上应用而言，Kubernetes和入口控制器提供了所有需要的功能，不需要任何更复杂的演进。

## Next Steps for More Complex Applications

*Add security, monitoring, and traffic management to improve control and visibility.*

When operations teams manage applications in production, they sometimes need deeper control and visibility. Sophisticated applications might exhibit complex network behavior, and frequent changes in production can introduce more risk to the stability and consistency of the app. It might be necessary to encrypt traffic between the components when running on a shared Kubernetes cluster.

Each requirement can be met using well‑understood techniques:

- To secure traffic between services, you can implement mutual TLS (mTLS) on each microservice, using [SPIFFE](https://spiffe.io/spiffe/)or an equivalent method.
- To identify performance and reliability issues, each microservice can export [Prometheus‑compliant metrics](https://prometheus.io/docs/instrumenting/exporters/) for analysis with tools such as Grafana.
- To debug those issues, you can embed [OpenTracing Tracers](https://opentracing.io/docs/overview/tracers/) into each microservice (multiple languages and frameworks are supported).
- To implement advanced load‑balancing policies, blue/green and canary deployments, and circuit breakers, you can tactically deploy proxies and load balancers.

![img](https://www.nginx.com/wp-content/uploads/2019/03/service-mesh_POTS.png)Individual microservices can be extended using **P**rometheus Exporters, **O**penTracing Tracers, mutual **T**LS, and **S**PIFFE (**POTS**). Proxies can be deployed ① to load balance individual services, or ② to provide a central Router Mesh.

Some of these techniques require a small change to each service – for example, burning certificates into containers or adding modules for Prometheus and OpenTracing. NGINX Plus can provide dedicated load balancing for critical services, with service discovery and API‑driven configuration for orchestrating changes. The [Router Mesh](https://www.nginx.com/blog/microservices-reference-architecture-nginx-router-mesh-model/) pattern in the NGINX Microservices Reference Architecture implements a cluster‑wide control point for traffic.

Almost every containerized application running in production today uses techniques like these to improve control and visibility.

## Why Then Do I Need a Service Mesh?

*If the techniques above are proven in production, what does a service mesh add?*

Each step described in the previous section puts a burden on the application developer and operations team to accommodate it. Individually, the burdens are light because the solutions are well understood, but the weight accumulates. Eventually, organizations running large‑scale, complex applications might reach a tipping point where enhancing the application service-by-service becomes too difficult to scale.

This is the core problem service mesh promises to address. The goal of a service mesh is to deliver the required capabilities in a standardized and transparent fashion, completely invisible to the application.

Service mesh technology is still new, with very few production deployments. Early deployments have been built on complex, home‑grown solutions, specific to each adopter’s needs. A more universal approach is emerging, described as the “sidecar proxy” pattern. This approach deploys Layer 7 proxies alongside every single service instance; these proxies capture all network traffic and provide the additional capabilities – mutual TLS, tracing, metrics, traffic control, and so on – in a consistent fashion.

![img](https://www.nginx.com/wp-content/uploads/2019/03/service-mesh_Nsidecars.png)In a service mesh, every container includes an embedded proxy which intercepts all ingress and egress traffic. The proxy handles encryption, monitoring and tracing on behalf of the service, and implements advanced traffic management.

Service mesh technology is still very new, and vendors and open source projects are rushing to make stable, functional, and easy-to-operate implementations. 2019 will almost certainly be the “[year of the service mesh](https://businesscomputingworld.co.uk/t/the-year-of-the-service-mesh-what-s-to-come-in-2019/1345)”, where this promising technology will reach the point where some implementations are truly production‑ready for general‑purpose applications.

## What Should I Do Now?

As of early 2019, it’s probably premature to jump forward to one of the early service mesh implementations, unless you have firmly hit the limitations of other solutions and need an immediate, short‑term solution. The immaturity and rapid pace of change in current service mesh implementations make the cost and risk of deploying them high. As the technology matures, the cost and risks will go down, and the tipping point for adopting service mesh will get closer.

![img](https://www.nginx.com/wp-content/uploads/2019/03/service-mesh_cost-to-operate.png)As the complexity of the application increases, service mesh becomes a realistic alternative to implementing capabilities service-by-service.

Do not let the lack of a stable, mature service mesh delay any initiatives you are considering today, however. As we have seen, Kubernetes and other orchestration platforms provide rich functionality, and adding more sophisticated capabilities can follow well‑trodden, well‑understood paths. Proceed down these paths now, using proven solutions such as ingress routers and internal load balancers. You will know when you reach the tipping point where it’s time to consider bringing a service mesh implementation to bear.