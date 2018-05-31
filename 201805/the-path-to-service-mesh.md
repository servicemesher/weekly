服务网格之路

![](https://ws1.sinaimg.cn/large/007ackX3ly1frux62q06sj333415oqv5.jpg)

When we talk to people about service mesh, there are a few questions we’re always asked. These questions range from straightforward questions about the history of our project, to deep technical questions on why we made certain decisions for our product and architecture.

当我们谈论服务网格的时候，有几个问题经常被提及。这些问题的范围覆盖从简单的了解服务网格的历史，到产品和架构相关的比较深入的技术问题。

To answer those questions we’ll bring you a three-part blog series on our Aspen Mesh journey and why we chose to build on top of Istio.

为了回答这些问题，通过 Aspen Mesh 之旅，我们带来三个主题的系列博文来讨论我们为什么选择了 Istio 。

To begin, I’ll focus on one of the questions I’m most commonly asked.

作为开始，我将重点讨论我最经常被问到的问题之一：

*Why did you decide to focus on service mesh and what was the path that lead you there?*

*为什么你选择服务网格，是什么原因促使你这样做？*

#### LineRate Systems: High-Performance Software Only Load Balancing

**LineRate 系统：高性能负载均衡软件**

The journey starts with a small Boulder startup called LineRate Systems and the acquisition of that company by F5 Networks in 2013. Besides being one of the smartest and most talented engineering teams I have ever had the privilege ofbeing part of, LineRate was a lightweight high-performing software-only L7 proxy. When I say high performance, I am talking about turning a server you already had in your datacenter 5 years ago into a high performance 20+ Gbps200,000+ HTTP requests/second fully featured proxy.

While the performance was eye-catching and certainly opened doors for our customers, our hypothesis was that customerswanted to pay for capacity, not hardware. That insight would turn out to be LineRate’s key value proposition. Thissimple concept would allow customers the ability to change the way that they consumed and deployed load balancers infront of their applications.

To fulfill that need we delivered a product and business model that allowed our customers to replicate the software asmany times as needed across COTS hardware, allowing them to get peak performance regardless of how many instances theyused. If a customer needed more capacity they simply upgraded their subscription tier and deployed more copies of theproduct until they reached the bandwidth, request rate or transaction rates the license allowed.

This was attractive, and we had some success there, but soon we had a new insight…

#### Efficiency Over Performance

It became apparent to us that application architectures were changing and the value curve for our customerswas changing along with them. We noticed in conversations with leading-edge teams that they were talking about conceptslike efficiency, agility, velocity, footprint and horizontal scale. We also started to hear from innovators in the spaceabout this new technology called Docker, and how it was going to change the way that applications and services weredelivered.

The more we talked to these teams and thought about how we were developing our own internal applications the more werealized that a shift was happening. Teams were fundamentally changing how they were delivering theirapplications, and the result was our customers were beginning to care less about raw performance and more aboutdistributed proxies. There were many benefits to this shift including reducing the failure domains of applications,increased flexibility in deployments and the ability for applications to store their proxy and network configuration ascode alongside their application.

At the same time containers and container orchestration systems were just starting to come on the scene, so we wentto work on delivering our LineRate product in a container with a new control plane and thinking deeply about how peoplewould be delivering applications using these new technologies in the future.

These early conversations in 2015 drove us to think about what application delivery would look like in the future…

#### That Idea that Just Won’t Go Away

As we thought more about the future of application delivery, we began to focus on the concept of policy and networkservices in a cloud-native distributed application world. Even though we had many different priorities and projects towork on, the idea of a changing application landscape, cloud-native applications and DevOps based delivery modelsremained in the forefront of our minds.

There just has to be a market for something new in this space.

We came up with multiple projects that for various reasons never came to fruition. We lovingly referred to them as v1.0,v1.5, and v2.0. Each of these projects had unique approaches to solving challenges in distributedapplication architectures (microservices).

So we thought as big as we could. A next-gen ADC architecture: a control plane that’s totally API-driven andseparate from the data plane. The data plane comes in any form you can think of: purpose-built hardware,software-on-COTS, or cloud-native components that live right near a microservice (like a service mesh). This infinitelyscalable architecture smooths out all tradeoffs and works perfectly for any organization of any size doing any kind ofwork. Pretty ambitious, huh? We had fallen into the trap of being all things to all users.

Next, we refined our approach in “1.5”, and we decided to define a policy language… The key was defining thatopen-source policy interface and connecting that seamlessly to the datapath pieces that get the work done. In a trulyopen platform, some of those datapath pieces are open source too. There were a lot of moving parts that didn’t all fallinto place at once; and in hindsight we should have seen some of them coming … The market wasn’t there yet, we didn’thave expertise in open source, and we had trouble describing what we were doing and why.

But the idea just kept burning in the back of our minds, and we didn’t give up…

For Version 2.0, we devised a plan that could help F5’s users who were getting started on their container journey.The technology was new and the market was just starting to mature, but we decided that customers would take three stepson their microservice journey:

1. *Experimenting* - Testing applications in containers on a laptop, server or cloud instance.
2. *Production Planning* - Identifying what technology is needed to start to enable developers to deploy container-basedapplications in production.
3. *Operating at Scale* - Focus on increasing the observability, operability and security of container applications toreduce the mean-time-to-discovery (MTTD) and mean-time-to-resolution (MTTR) of outages.

We decided there was nothing we could do for experimenting customers, but for production planning, we could create anopen source connector for container orchestration environments and BIG-IP. We called this the BIG-IP ContainerConnector, and we were able to solve existing F5 customers’ problems, and start talking to them about the nextstep in their journey. The container connector team continues to this day to bridge the gap between ADC-as-you-know-itand fast-changing container orchestration environments.

We also started to work on a new lightweight containerized proxy called the Application Services Proxy, or ASP.Like Linkerd and Envoy, it was designed to help microservices talk to each other efficiently, flexibly and observably.Unlike Linkerd and Envoy, it didn’t have any open source community associated with it. We thought about ouropen source strategy and what it meant for the ASP.

At the same time, a change was taking place within F5…

#### Aspen Mesh - An F5 Innovation

As we worked on our go to market plans for ASP, F5 changed how it invests in newtechnologies and nascent markets through incubation programs. These two events, combined with the explosive growth inthe container space, led us to the decision to commit commit to building a product on top of an existing open sourceservice mesh. We picked Istio because of its attractive declarative policy language, scalable control-plane architectureand other things that we’ll cover in more depth as we go.

With a plan in place it was time to pitch our idea for the incubator to the powers that be. Aspen Mesh is the result ofthat pitch and the end of one journey, and the first step on a new one…

Parts two and three of this series will focus on why we decided to use Istio for our service mesh core and what you canexpect to see over the coming months as we build the most fully supported enterprise service mesh on the market.