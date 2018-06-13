# [服务网格和Cookpad]()

这个原文是5月初发表的[原文](http://techlife.cookpad.com/entry/2018/05/08/080000)的翻译。为了弥补这篇文章的背景，Cookpad 是一家拥有200多种产品开发的中型科技公司，拥有10多支团队，每月平均用户数量达到9000万。[https://www.cookpadteam.com/](https://www.cookpadteam.com/)

---

你好，这是来自开发人员生产力团队的[Taiki](https://github.com/taiki45/)。目前，我想介绍一下在 Cookpad 上构建和使用服务网格所获得的知识。

对于服务网格本身，我认为您将对以下文章，公告和教程有完整的体验：

* [https://speakerdeck.com/taiki45/observability-service-mesh-and-microservices](https://speakerdeck.com/taiki45/observability-service-mesh-and-microservices)
* [https://buoyant.io/2017/04/25/whats-a-service-mesh-and-why-do-i-need-one/](https://buoyant.io/2017/04/25/whats-a-service-mesh-and-why-do-i-need-one/)
* [https://blog.envoyproxy.io/service-mesh-data-plane-vs-control-plane-2774e720f7fc](https://blog.envoyproxy.io/service-mesh-data-plane-vs-control-plane-2774e720f7fc)
* [https://istioio.io/docs/setup/kubernetes/quick-start.html](https://istioio.io/docs/setup/kubernetes/quick-start.html)
* [https://www.youtube.com/playlist?list=PLj6h78yzYM2P-3-xqvmWaZbbI1sW-ulZb](https://www.youtube.com/playlist?list=PLj6h78yzYM2P-3-xqvmWaZbbI1sW-ulZb)

## 我们的目标

我们引入了一个服务网格来解决故障排除，容量规划和保持系统可靠性等操作问题。尤其是：

* 降低服务的管理成本
* 可观察性的改进 \(分别参考了 [ Twitter ](https://blog.twitter.com/engineering/en_us/a/2013/observability-at-twitter.html) 和 [Medium的博客](https://medium.com/@copyconstruct/monitoring-and-observability-8417d1952e1c)\)
* 建立更好的故障隔离机制

就第一个问题而言，随着规模的扩大，存在难以掌握哪个服务和哪个服务正在进行通信，某个服务的失败是哪里传播导致的问题。我认为这个问题应该通过集中管理服务在哪里和服务在哪里连接的相关信息来解决。

对于第二个，我们进一步挖掘了第一个，这是一个问题，我们不知道一个服务与另一个服务之间的通信状态是否容易。例如，RPS，响应时间，成功/失败状态的数量，超时，断路器的状态等。在两个或更多个服务引用某个后端服务的情况下，解析来自代理或负载均衡器的度量后端服务不足，因为它们未被请求源服务标记。

对于第三个问题，“故障隔离配置尚未成功设置”是一个问题。那时，在每个应用程序中使用库，超时，重试，断路器的设置完成。但要知道什么样的设置，有必要单独查看应用程序代码。没有清单和情况掌握，难以持续改进这些设置。另外，因为与故障隔离有关的设置应该不断改进，所以最好是可测试的，并且我们需要这样一个平台。

为了解决更先进的问题，我们还构建了gRPC 基础设施建设，配送跟踪处理委托，流量控制部署方式多样化，认证授权网关等功能。这个区域将在稍后讨论。

## 当前状态

Cookpad 中的服务网格使用 Envoy 作为数据平面并创建了我们自己的控制平面。尽管我们最初考虑安装已经作为服务网格实现的[Istio](https://istio.io/)，但几乎 [Cookpad](https://istio.io/) 中的所有应用程序都使用名为 AWS ECS 的容器管理服务进行操作，因此与 Kubernetes 合作的优点是有限的。考虑到我们想实现的目标以及 Istio 软件本身的复杂性，我们选择了我们自己的控制平面的路径，该平面可以从小型起步。

此次实施的服务网格的控制面部分由几个组件组成。我将解释每个组件的角色和操作流程：

* 集中管理服务网格配置的存储库。
* 使用名为 [kumonos](https://github.com/taiki45/kumonos) 的gem，将生成[Envoy xDS API](https://github.com/envoyproxy/data-plane-api/blob/5ea10b04a950260e1af0572aa244846b6599a38f/API_OVERVIEW.md) 响应 JSON
* 将生成的响应 JSON 放置在 Amazon S3 上，并将其用作 Envoy 的 xDS API

该设置在中央存储库中进行管理的原因是，

* 我们希望随时跟踪更改历史记录并在稍后跟踪它
* 我们希望能够检查各个组织（如SRE团队）的设置更改

关于负载平衡，我最初是由 Internal ELB 设计的，但 gRPC 应用程序的基础架构也符合要求\(我们的 gRPC 应用程序已经在生产环境中使用此机制\)，我们使用 SDS（服务发现服务）API \(简单地使用内部 ELB（NLB或TCP模式CLB）的服务器端负载均衡由于不平衡的平衡而在性能方面具有缺点，并且在可获得的度量方面也是不够的\) 准备了客户端负载平衡。我们在 ECS 任务中部署了一个侧车容器，用于对应用程序容器执行健康检查并在 SDS API 中注册连接目标信息。

![](https://ws1.sinaimg.cn/large/61411417ly1fs7pzdtqd9j20n60dq40a.jpg )

度量标准的配置如下所示：

* 将所有指标存储到 Prometheus
* 发送标签的度量来 [statsd\_exporter](https://github.com/prometheus/statsd_exporter) 使用[dog\_statsd sink](https://www.envoyproxy.io/docs/envoy/v1.6.0/api-v2/config/metrics/v2/stats.proto#config-metrics-v2-dogstatsdsink) ECS 容器主机实例运行 （起初我将它作为我们自己的扩展实现，但稍后我发送了一个[补丁](https://github.com/envoyproxy/envoy/pull/2158)）

* 所有指标都包含通过 [固定字符串标签](https://www.envoyproxy.io/docs/envoy/v1.6.0/api-v2/config/metrics/v2/stats.proto#config-metrics-v2-statsconfig) 的应用程序 ID 来标识每个节点 (这个是我们的另一个[补丁](https://github.com/envoyproxy/envoy/pull/2357))
* 普罗米修斯使用 [EC2 SD] (https://prometheus.io/docs/prometheus/latest/configuration/configuration/) 拉动度量信息
* 要管理 Prometheus 的端口，我们在 statsd\_exporter 和 Prometheus 之间使用 [exporter\_proxy](https://github.com/rrreeeyyy/exporter_proxy)
* 使用 Grafana 和 [Vizceral](https://medium.com/netflix-techblog/vizceral-open-source-acc0c32113fe) 进行度量指标

如果应用程序进程在不使用 ECS 或 Docker 的情况下直接在 EC2 实例上运行，Envoy 进程作为守护进程直接在实例中运行，但体系结构几乎相同。有一个原因是没有将 Prometheus 直接设置为 Envoy ，因为我们仍然无法从 Envoy 的 Prometheus 兼容端点中 [提取直方图度量](https://github.com/envoyproxy/envoy/issues/1947)。由于这将在未来得到改善，我们计划在当时消除 stasd\_exporter。

![](https://ws1.sinaimg.cn/large/61411417ly1fs7pv3rapdj20sg0qvgpb.jpg )

在 Grafana 上，仪表板和 Envoy 的整个仪表板都为每项服务做好准备，例如上游 RPS 和超时发生。我们还将准备一个服务x服务维度的仪表板。

每个服务仪表板：

![](https://ws1.sinaimg.cn/large/61411417ly1fs7pv4dqikj20sg0mp11e.jpg )

例如，上游电量不足时的断路器相关指标：

![](https://ws1.sinaimg.cn/large/61411417ly1fs7pv4kw6vj20i40d9q41.jpg )

Envoy 的仪表板：

![](https://ws1.sinaimg.cn/large/61411417ly1fs7pv4rqrij20sg0qa49n.jpg )

使用 Netflix 开发的 Vizceral 可视化服务配置。为了实现，我们开发了 [promviz](https://github.com/nghialv/promviz) 和 [promviz-front](https://github.com/mjhd-devlion/promviz-front)[\* 8的](#f-3ae4bcd1)fork。由于我们仅为某些服务介绍它，因此当前显示的节点数量很少，但我们提供了以下仪表板。

每个地区的服务配置图，RPS，错误率：

![](https://ws1.sinaimg.cn/large/61411417ly1fs7pv47xzjj20sg0gxdjd.jpg )

特定服务的下游/上游：

![](https://ws1.sinaimg.cn/large/61411417ly1fs7pv3xymcj20sg0i2acs.jpg)

作为服务网格的一个子系统，我们部署了一个网关，用于从我们办公室的开发人员计算机访问登台环境中的 gRPC 服务器应用程序[\* 9](#f-81abbe53)。它是通过将 SDS API 和 Envoy 与管理称为[hako-console的](http://techlife.cookpad.com/entry/2018/04/02/140846)内部应用程序的软件相结合而构建的。

* 网关应用程序（Envoy）向网关控制器发送 xDS API 请求
* 网关控制器从hako控制台获取临时环境中的 gRPC 应用程序列表，并基于该响应返回路径发现服务/集群发现服务 API 响应
* 网关应用根据响应从 SDS API 获取实际连接目的地
* 从开发人员手中引用 AWS ELB 网络负载平衡器，网关应用程序执行路由

![](https://ws1.sinaimg.cn/large/61411417ly1fs7pv42jzej20sg0mmtaz.jpg)

## 结果

引入服务网格最显着的是它能够抑制临时残疾的影响。有许多交通服务之间的多个合作部分，到现在为止，200多个与网络相关的琐碎错误[\* 10](#f-a7617164)一直在不断地在一小时内发生的[\* 11](#f-2cb8e98a)，它减少到约是否能在一周或不拿出服务网格的正确重试设置。

从监测的角度来看，各种指标已经出现，但由于我们只是针对某些服务介绍了这些指标，并且由于推出日期我们还没有达到全面使用，我们预计将来会使用它。在管理方面，当服务之间的连接变得可见时，我们很容易理解我们的系统，因此我们希望通过将服务引入所有服务来避免忽视和忽略对象。

## 将来的计划

#### 迁移到 v2 API，转换到 Istio

由于 xDS API 的初始设计情况和使用 S3 作为后端交付的要求，xDS API 一直在使用 v1，但由于 v1 API 已被弃用，因此我们计划将其移至 v2。与此同时，我们正在考虑将控制飞机移至Istio。另外，如果我们要制造我们自己的控制平面，我们将使用 [go-control-plane](https://github.com/envoyproxy/go-control-plane) 来制作 LDS / RDS / CDS / EDS API[\* 12](#f-3ef2cbdf)。

#### 替换反向代理

到目前为止，Cookpad 使用 NGINX 作为反向代理，但是我们考虑到 NGINX 和 Envoy 在内部技术实现，gRPC 通信和采集度量方面的差异，我们将考虑用 Envoy 替换 NGINX 的反向代理和边缘代理。

#### 流量控制

随着我们转向客户端负载均衡并取代反向代理，我们将能够通过操作 Envoy 更方便的处理流量，所以我们将能够实现金丝雀部署，流量转移和请求镜像。

#### 故障注入

这是一个故意在正确管理的环境中注入延迟和故障的机制，并测试实际服务组是否正常工作。Envoy 有各种功能[\* 13](#f-794018e7)。

#### 在数据平面层上执行分布式跟踪

在Cookpad中，AWS X-Ray 被用作分布式追踪系统 [\* 14](#f-fdcfb94c)。目前，我们将分布式跟踪功能作为一个库来实现，但我们计划将其移至数据平面并在服务网格层实现。

#### 身份验证授权网关

这是为了仅在接收用户请求的最前端服务器进行认证和授权处理，随后的服务器将使用结果。以前，它不完全是作为一个图书馆来实施的，但是通过转向数据平台，我们可以获得过程模型的优点。

## 最后

我们已经介绍了Cookpad中服务网格的现状和未来。许多功能已经可以很容易地实现，并且由于将来可以通过服务网格层完成更多的工作，因此强烈建议每个微服务系统。


[\* 7](fn-ae4435b7)：[https://github.com/envoyproxy/envoy/issues]()

[\* 8](fn-3ae4bcd1)：为了方便用NGINX交付并符合Cookpad中的服务组合

[\* 9](fn-81abbe53)：假设使用客户端负载平衡进行访问，我们需要一个组件来解决它。

[\* 10](fn-a7617164)：与流量相比，这个数字非常小。

[\* 11](fn-2cb8e98a)：尽管在一些 partes 中设置了重试。

[\* 12](fn-3ef2cbdf)：[https](https://github.com/envoyproxy/data-plane-api/blob/5ea10b04a950260e1af0572aa244846b6599a38f/API_OVERVIEW.md#apis)：[//github.com/envoyproxy/data-plane-api/blob/5ea10b04a950260e1af0572aa244846b6599a38f/API\_OVERVIEW.md\#apis](https://github.com/envoyproxy/data-plane-api/blob/5ea10b04a950260e1af0572aa244846b6599a38f/API_OVERVIEW.md#apis)

[\* 13](fn-794018e7)：[https](https://www.envoyproxy.io/docs/envoy/v1.6.0/configuration/http_filters/fault_filter.html):[//www.envoyproxy.io/docs/envoy/v1.6.0/configuration/http\_filters/fault\_filter.html](https://www.envoyproxy.io/docs/envoy/v1.6.0/configuration/http_filters/fault_filter.html)

[\* 14](fn-fdcfb94c)：[http](http://techlife.cookpad.com/entry/2017/09/06/115710):[//techlife.cookpad.com/entry/2017/09/06/115710](http://techlife.cookpad.com/entry/2017/09/06/115710)
