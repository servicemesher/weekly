# 服务网状结构激化了容器网络

> 原文链接：https://searchitoperations.techtarget.com/feature/Service-mesh-architecture-radicalizes-container-networking
>
> 作者：[Beth Pariseau](https://www.techtarget.com/contributor/Beth-Pariseau)
>
> 译者：殷龙飞

## 容器化是IT行业最喜欢的超级英雄，因此容器在服务网格中具有强大的伙伴关系是唯一的选择。他们一起对抗网络管理混乱。

这篇文章也可以在高级版中找到。 [现代堆栈：Kubernetes sidecar 是否能提供容器般的快乐？](https://searchitoperations.techtarget.com/ezine/Modern-Stack/Will-the-Kubernetes-sidecar-deliver-container-happiness)



![](https://ws1.sinaimg.cn/large/61411417ly1fsz4uo3uqcj20xc0b4aby.jpg)



[Beth Pariseau](https://www.techtarget.com/contributor/Beth-Pariseau)

高级新闻作家



容器和微服务产生了一种称为服务网格的新型网络架构范例，但 IT 行业观察人士对它是否会看到广泛的企业用途持不同意见。



服务网格体系结构使用一个代理，该代理称为附加到每个应用程序容器，虚拟机或容器编排 pod 的 *sidecar 容器*，具体取决于所使用的服务网格的类型。然后，该代理可以连接到集中式控制平面软件，这些软件收集细粒度的网络遥测数据，应用网络管理策略或更改代理配置，建立并执行网络安全策略。

IT系统中的服务网格体系结构还处于初期阶段，但与集装箱一样，其突出地位一直很快。在 2017 年 12 月云原生计算基金会（CNCF）的 KubeCon 和 CloudNativeCon 上，服务网格已经绕过容器成为[尖端 DevOps 商店中](https://searchitoperations.techtarget.com/essentialguide/Use-these-DevOps-examples-to-reimagine-an-IT-organization)最热门的主题。

“我们经常发现自己希望构建应用软件，但我们实际上在做的是一遍又一遍地编写相同的代码来解决某些实际上非常困难的计算机科学问题，这些问题应该被考虑到某种通用接口中”，微服务监控创业公司 LightStep 首席执行官 Ben Sigelman 在 KubeCon 的服务网格主题演讲中表示。

“服务网格可以帮助发现服务，互连这些服务，断路由，负载均衡，......安全和身份验证” , Sigelman说，他是前谷歌工程师，OpenTracing 的创建者，OpenTracing 是开源的，提供不依赖供应商的 API。 

### 服务网格简史

最早版本的 sidecar 代理技术在 2016 年初开始出现在网络规模的商店，如谷歌和推特，微服务管理需要对网络进行新的思考。与传统的单体应用程序不同，[微服务](https://searchmicroservices.techtarget.com/definition/microservices)依靠外部网络来沟通和协调应用程序功能。这些微服务通信需要密切监控，有时需要大规模重新配置。

用于使微服务网络管理自动化的最早技术依赖于库，作为应用程序代码的一部分进行部署，[如 Netflix 的 Hystrix](https://github.com/Netflix/Hystrix)。因此，开发人员需要进行网络管理。这些库也必须用特定环境中使用的每种应用程序语言编写。这提出了一个难题，因为[微服务精神](https://searchmicroservices.techtarget.com/answer/How-will-microservices-development-benefit-enterprise-architecture)的一个主要原则是小团队可以自由地使用任何语言进行独立的服务管理。

> 大多数认为自己正在做微服务的组织并没有真正做到真正的微服务。   **安妮托马斯**分析师，Gartner

在 2016 年初，在 Twitter 上实施了第一批微服务的工程师成立了 Buoyant 公司，该公司采用 sidecar 代理方法替代应用程序库。Buoyant 在 2016 年年中创造了术语*服务网格*，其最初的服务网格产品 Linkerd 使用 Java 虚拟机（JVM）作为 sidecar，这种设计将网络管理负担从应用程序开发人员转移出来，并支持对多语言的集中管理应用网络。到目前为止，Linkerd 是主流企业 IT 商店中唯一上生产环境的服务网格体系结构。使用的客户包括 Salesforce、PayPal、Credit Karma、Expedia 和 AOL。

当 Linkerd 刚刚站稳了脚跟时，[Docker 容器](https://searchitoperations.techtarget.com/definition/Docker)和 [Kubernetes 容器编排](https://searchitoperations.techtarget.com/definition/Google-Kubernetes)将 [Buoyant](https://searchitoperations.techtarget.com/definition/Google-Kubernetes) 工程师送会起点。终于在2017 年 12 月，该公司发布了 Conduit，一种基于轻量级容器代理的服务网格体系结构，而不是 Linkerd 的资源沉重的 JVM。它专门用于与 [Go](https://searchitoperations.techtarget.com/tip/Googles-Go-language-seeks-DevOps-middle-ground) 和 [Rust](https://research.mozilla.org/rust/) 应用程序语言组合使用的 Kubernetes 。

Kubernetes 社区正在为 Go 编写轻量级服务，可能需要 20 MB 或 50 MB 的内存才能运行，而 Linkerd 的 JVM可能会占用 200 MB 的内存，对于 Kubernetes 爱好者来说这是一个矛盾点，William Morgan说 ，他是 Buoyant 的联合创始人兼首席执行官。

Morgan 说：“它需要花费大量内存这不是最理想 ，特别是当价值主张是它将成为开发人员不必担心的底层基础架构的一部分时。

但就在 2017 年初 Buoyant 工程师开始重新考虑其服务网格体系结构时，Kubernetes 的创造者谷歌和重量级技术公司 IBM 联手  Lyft 公司的 Envory 创建了  [Istio](https://searchmicroservices.techtarget.com/news/450419875/IBM-Google-Lyft-launch-Istio-open-source-microservices-platform)。鉴于其支持者的声誉和谷歌内部管理大规模基于容器的微服务的经验，这种基于容器的服务网格引起了业界的广泛关注。Google 基于其内部的服务控制工具向 Istio 提供控制平面软件，而 IBM 则添加了控制平面工具 Amalgam8。Istio 是基于 Lyft 的 Envoy sidecar 代理，该公司是为了控制平面接收命令而建立的。它可以动态读取到 sidecar 的配置更新，而无需重启 。

![](https://ws1.sinaimg.cn/large/61411417ly1fsz4wgsjvkj20m80oomy3.jpg)

Istio 的支持者正在与 Kubernetes 的家园 CNCF 进行长期管理谈判。他们计划在 2018 年第三季度发布 1.0 版本的产品。

到目前为止，Linkerd 和 Istio 已经成为这个新兴市场中最具影响力的公司，但是还有很多服务网格体系结构项目正在进行中，包括开源和专有选项。这些项目中有许多是基于 Envoy sidecar。Nginx 基于其 Nginx Plus代理引入[了自己的集中式管理控制平面](https://itknowledgeexchange.techtarget.com/open-source-insider/nginx-gets-granular-on-managed-microservices/)。其他早期的服务网格希望包括 Turbine Labs 的 Houston，Datawire 的 Ambassador，Heptio 的 Contour，Solo.io 的 Gloo 和 Tigera 的 CNX。

### 谁需要服务网格？

现在判断服务网络架构在主流企业IT商店中的普及程度还为时过早，这些商店IT商店不适用于Twitter或Google 。

Gartner 分析师 Anne Thomas 表示，对于以有限方式使用容器的组织，现有 API 网关和 Kubernetes 或 PaaS 软件（如 Docker Enterprise Edition 或 Cloud Foundry）的服务发现和网络管理功能可能会提供足够的微服务支持。

“大多数认为他们正在做微服务的组织并没有真正做到真正的微服务 “，Thomas 说。“我不相信真正的微服务将成为传统企业中的主流。”

>  \[服务网格\]允许您以集中的方式推动流量，这种方式在许多不同的环境和技术中保持一致 ，我觉得这在任何规模上都很有用。 **Zack Angelo** BigCommerce 平台工程总监 

对 Thomas 来说，真正的微服务是尽可能独立的。每个服务处理一个单独的方法或域功能; 使用自己的独立数据存储; 与其他微服务依靠基于异步事件的通信; 并允许开发人员设计，开发，测试，部署和替换这个单独的功能，而无需重新部署应用程序的任何其他部分。

“很多主流公司并不一定愿意花很多时间和金钱来投入他们的应用架构”，Thomas 争辩道。“他们仍然在以更粗粒度的方式做事，而且他们不会使用网格，至少在网格作为他们正在使用的服务构建到平台之前，或者直到我们获得品牌 \- 新的发展框架“。

服务网格体系结构的一些早期采用者并不认为需要大量的微服务才能从该技术中受益 。

“它可以让你以集中的方式推动流量，这种流量在许多不同的环境和技术中是一致的，我觉得这在任何规模上都很有用”，电子商务公司 BigCommerce 的平台工程主管 Zack Angelo 说。德克萨斯州奥斯汀，使用 Linkerd 服务网格”。“即使你有 10 到 20 种服务，这也是非常有用的功能”。

Angelo 说，传统的网络管理概念，例如负载均衡器，无法按微小的百分比把流量路由到某些节点 ，以便进行[金丝雀或蓝/绿应用程序的部署](https://searchitoperations.techtarget.com/tip/Improve-application-rollout-planning-with-advanced-options)。传统的网络监控工具也不提供服务网格提供的那种粒度的遥测数据，这使得 Angelo  能够跟踪 99% 的应用程序延迟中的微小异常，其重要性在服务网格中被放大。

Linkerd 的负载均衡模式使用了一种称为*指数加权移动平均*的技术，以便当服务网格跨主机分配网络流量时，它会考虑下游服务响应的速度，然后将流量路由到服务性能最佳的地方，而不是传统循环负载平衡技术。

>  他们拥有实时数据并且为每位用户个性化体验都很重要。Google 的 Istio 产品管理总监  **Jennifer Lin**

“我们的应用分布在多个数据中心，很高兴将技术内置到我们的负载均衡器中，这将自动知道并选择最快的网络路径 ”。Angelo 说。“从故障转移的角度来看，这对我们非常有趣”。

这并不是说服务网络没有权衡 ，特别是当涉及 IT 运营人员不熟悉高级网络概念的管理复杂性时。Angelo 表示，如果管理不当，集中式控制平面可能会成为自己的单点故障，尽管企业可以通过在其服务网格设计中增加弹性来降低这种风险。

“如果在服务发现中发生了某些事情，向 Linkerd 节点提供陈旧的数据或其他内容，并且负载均衡池中存在错误的主机，则即使服务发现信息不正确，Linkerd 失败算法也会将其从池中取出，这真是太好了“，Angelo 说。

其他公司看好 Istio 的集中化网络监控功能，计划在 Istio 进入 GA 状态后跟进。

“我们仍然有 PHP，Node 和 Go 中的\[应用程序代码\]，以及三种不同的方式来收集日志，监控服务和正常运行时间 ”，Harrison Harnisch说道，他是一名位于芝加哥的Buffer工作人员，一个分布式社交媒体管理平台美国各地的员工 。”但如果我们能够通过服务网络获得所有内容，我们就可以使用相同的模式进行日志记录，并构建模板仪表板以便跨团队共享，这在现在很难做到" 。 

### Istio 的创造者研究网状展望

即使在银行业等传统行业中，开发人员也在创建复杂的面向消费者的应用程序，这些应用程序看起来更像是Google 这样的高规模网络应用程序。

“重要的是，他们有实时数据，并且他们为每个用户提供个性化体验”，谷歌 Istio 产品管理总监 Jennifer Lin 说。“这需要一个更细粒度的服务集，允许这些创新的应用程序以安全的方式以极低的延迟大规模地做事 ” 。

IBM 工程师 Daniel Berg 说，精细的流量路由和安全策略也将成为 IBM 推出的 Istio 混合云概念的关键组成部分，并且将有必要管理私有云和公共云中的微服务。

“客户将需要一个网格来帮助组织和管理传统和云原生应用程序之间转换所带来的复杂性 ”， Berg 说。“如果您开始使用任何网格作为应用程序的一部分，如果您尝试将其移植到另一个未使用它的提供程序，但它可能会运行，您会得到非常不同的行为，这可能是意想不到的并且是不可取的“。

但 Envoy 的高级软件工程师 Matt Klein 的表示，主流企业最有可能等到服务网状体系结构的特征成为[公共云容器作为服务和PaaS产品的一部分](https://searchitoperations.techtarget.com/tip/Container-as-a-service-providers-compete-with-distinct-strategies)，这与 Gartner 的 Thomas 的预测相呼应 。

“你可以对它进行成像的方式可以像 AWS Fargate 那样工作，他们会在每个用户功能或容器旁自动注入一个像Envoy 这样的代理，而且用户只需要了解这些功能而无需关心它们的实际情况实施“ ，Klein说。“他们会获得服务网格功能，但对他们而言，它的服务网格并不重要 ”。

Klein 说，也有人猜测向这种服务过渡需要多长时间。

Klein 说：“从大多数事情发生在某种类型的公共云中时，我们可能需要10到20年的时间 ”。 “像\[微软\] Azure，\[Google云平台\]和亚马逊这样的企业都是百年企业，我们正处于这个阶段的最初阶段”。