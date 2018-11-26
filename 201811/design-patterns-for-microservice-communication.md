---
original: https://blog.jdriven.com/2018/11/transcoding-grpc-to-http-json-using-envoy/
translator: malphi
reviewer: rootsongjc
title: "微服务通信的设计模式"
description: "本文用实例讲解了如何利用Envoy将gRPC转码为HTTP/JSON"
categories: "translation"
tags: ["Envoy"]
date: 2018-11-19
---

# 微服务通信的设计模式

在我的上一篇博客中，我谈到了[微服务的设计模式](https://dzone.com/articles/design-patterns-for-microservices)。现在，我想更深入地探讨微服务体系结构中最重要的模式:微服务之间的相互通信。我仍然记得我们过去开发单一应用程序的时候;通讯是一项艰巨的任务。在那个世界中，我们必须小心的设计数据库表和对象模型映射之间的关系。现在，在微服务世界中，我们已经将它们分解为独立的服务，并在它们周围创建了网格来彼此通信。让我们来谈谈迄今为止为解决这个问题而发展起来的所有沟通方式和模式。

许多架构师已经将微服务之间的通信划分为同步和异步通讯。让我们一个一个来介绍。

## 同步

当我们说同步时，它意味着客户机向服务器发出请求并等待其响应。线程将被阻塞，直到它接收到通信返回。实现同步通信最相关的协议是HTTP。HTTP可以通过REST或SOAP实现。最近，REST在微服务方面发展迅速，并赢得了SOAP的支持。对我来说，两者都很好用。

现在让我们讨论同步中的不同流/用例、我们面临的问题以及如何解决它们。

1. 让我们从一个简单的例子开始。您需要一个服务A调用服务B并等待对在线数据的响应。这是实现同步的一个很好的候选，因为不涉及很多下游服务。如果使用多个实例，除了负载平衡之外，您不需要为这个用例实现任何复杂的设计模式。
2. ![sync flow](https://ws2.sinaimg.cn/large/006tNbRwly1fxlg5e91x1j30fc04yt8l.jpg)
3. 现在，让我们把它变得更复杂一点。服务A为实时数据调用多个下游服务，如服务B、服务C和服务D。

- 服务B、服务C和服务D都必须按顺序调用——当服务相互依赖以检索数据或功能通过这些服务执行一系列事件时，就会出现这种场景。
- 服务B、服务C和服务D可以并行调用——这种场景将在服务彼此独立或服务A可能扮演协调者角色时使用。
- ![sync flow2](https://ws1.sinaimg.cn/large/006tNbRwly1fxlgbk5vfbj30g609rwei.jpg)

这个场景在进行通信时带来了复杂性。让我们一个一个地讨论。

### **紧密耦合**

服务A将与每个服务B、C和D紧密耦合。它必须知道每个服务的端点（endpoint）和凭据（credentials）。

**解决方案：** [服务发现模式](https://www.rajeshbhojwani.co.in/2018/11/design-patterns-for-microservices.html) 就是用来解决这类问题的。它通过提供查找功能来帮助分离消费者和生产者应用。服务B、C和D可以将它们自己注册为服务。服务发现可以在服务器端实现，也可以在客户端实现。对于服务器端，我们有AWS ALB和NGINX工具，它们接受来自客户机的请求、发现服务并将请求路由到指定位置。

对于客户端，我们有Spring Eureka discovery服务。使用Eureka的真正好处是它在客户端缓存了可用的服务信息，所以即使Eureka服务器宕机一段时间，它也不会成为一个单点故障。除了Eureka, etcd和consul等其他服务发现工具也得到了广泛的应用。

### **分布式系统**

如果服务B，C，D有多个实例，它们需要知道如何去负载均衡。

**解决方案：** 负载均衡通常与服务发现携手出现。对于服务器端负载平衡，可以使用AWS ALB，对于客户端，可以使用Ribbon或Eureka。

### **验证/过滤/处理协议**

如果服务B、C和D需要保护并需要身份验证，我们只需要过滤这些服务的某些请求，如果服务A和其他服务理解不同的协议。

**解决方案：** [API 网关模式](http://www.rajeshbhojwani.co.in/2018/11/design-patterns-for-microservices.html) 有助于解决这些问题。它可以处理身份验证、过滤和将协议从AMQP转换为HTTP或其他协议。它还可以帮助启用分布式日志记录、监视和分布式跟踪等可观察性指标（metrics）。Apigee、Zuul和Kong是一些可以用于此的工具。请注意，如果服务B、C和D是可管理的API的一部分，我建议使用这种模式，否则使用API网关就太过了。进一步阅读服务网格作为替代解决方案。

### **处理失败**

如果任何服务B、C或D宕机，如果服务A仍然可以使用某些特性来响应客户端请求，则必须相应地对其进行设计。另一个问题是：假设服务B宕机，所有请求仍然在调用服务B，并且由于它没有响应而耗尽了资源。这会使整个系统宕机，服务A也无法向C和D发送请求。

**Solution:** [熔断器](http://www.rajeshbhojwani.co.in/2018/11/design-patterns-for-microservices.html) and [隔离 ](https://docs.microsoft.com/en-us/azure/architecture/patterns/bulkhead)模式有助于解决这些问题。断路器模式识别下游服务是否停机一段时间，并断开电路以避免向其发送调用。如果服务已经恢复并关闭电路以继续对它的调用，它将在定义的时间段之后再次尝试检查。这确实有助于避免网络阻塞和耗尽资源。隔离壁有助于隔离用于服务的资源，并避免级联故障。Spring Cloud Hystrix也做同样的工作。它适用于断路器和舱壁模式。

### **微服务间网络通信**

[API Gateway](http://www.rajeshbhojwani.co.in/2018/11/design-patterns-for-microservices.html) 通常用于管理API，它处理来自ui或其他消费者的请求，并对多个微服务进行下游调用并作出响应。但是，当一个微服务想要调用同一组中的另一个微服务时，API网关就没有必要了，它并不是为了这个目的而设计的。最终，单个微服务将负责进行网络通信、进行安全身份验证、处理超时、处理故障、负载平衡、服务发现、监视和日志记录。对于微服务来说，开销太大了。

**解决方案：** 服务网格模式有助于处理此类NFRs。它可以卸载我们前面讨论的所有网络功能。这样，微服务就不会直接调用其他微服务，而是通过这个服务网格，它将处理所有的通信。这种模式的美妙之处在于，现在您可以专注于用任何语言(如Java、NodeJS或Python)编写业务逻辑，而不必担心这些语言是否支持实现所有网络功能。Istio和Linkerd解决了这些需求。我唯一不喜欢Istio的地方是，它目前仅限于Kubernetes。

## 异步

When we talk about asynchronous communication, it means the client makes a call to the server, receives acknowledgment of the request, and forgets about it. The server will process the request and complete it.

Now let's talk about when you need the asynchronous style. If you have an application which is read-heavy, the synchronous style might be a good fit, especially when it needs live data. However, when you have write-heavy transactions and you can't afford to lose data records, you may want to choose asynchronous because, if a downstream system is down and you keep sending synchronous calls to it, you will lose the requests and business transactions. The rule of thumb is to never ever use async for live data read and never ever use sync for business-critical write transactions unless you need the data immediately after write. You need to choose between availability of the data records and strong consistency of the data.

There are different ways we can implement the asynchronous style:

### **Messaging**

In this approach, the producer will send the messages to a message broker and he consumer can listen to the message broker to receive the message and process it accordingly. There are two patterns within messaing: one-to-one and one-to-many. We talked about some of the complexity synchronous style brings, but some of it is eliminated by default in the messaging style. For example, service discovery becomes irrelevant as the consumer and producer both talk only to the message broker. Load balancing is handled by scaling up the messaging system. Failure handling is in-built, mostly by the message broker. RabbitMQ, ActiveMQ, and Kafka are the best-known solutions in cloud platforms for messaging.

![Image title](https://dzone.com/storage/temp/10665685-async-msg.png)

### **Event-Driven**

The event-driven method looks similar to messaging, but it serves a different purpose. Instead of sending messages, it will send event details to the message broker along with the payload. Consumers will identify what the event is and how to react to it. This enables more loose coupling. There are different types of payloads that can be passed:

- Full payload — This will have all the data related to the event required by the consumer to take further action. However, this makes it more tightly coupled.
- Resource URL — This will be just a URL to a resource that represents the event.
- Only event — No payload will be sent. The consumer will know based on on the event name how to retrieve relevant data from other sources, like databases or queues.

![Image title](https://dzone.com/storage/temp/10665753-async-event.png)

There are other styles, like choreography style, but I personally don't like that. It is too complicated to be implemented. This can only be done with the synchronous style.

That's all for this blog. Let me know your experience with microservice-to-microservice communication.