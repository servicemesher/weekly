# 第3部分：基于Ambassador API网关实现Java速率限制服务

> 原文链接：https://blog.getambassador.io/implementing-a-java-rate-limiting-service-for-the-ambassador-api-gateway-e09d542455da
>
> 作者：Daniel Bryant
>
> 译者：戴佳顺

基于Kubernetes云原生的[Ambassador API](https://www.getambassador.io/)网关所提供的速率限制功能是完全可定制的，其允许任何实现gRPC服务端点的服务自行决定是否需要对请求进行限制。本文在先前[第1](https://blog.getambassador.io/rate-limiting-a-useful-tool-with-distributed-systems-6be2b1a4f5f4)和[第2部分](https://blog.getambassador.io/rate-limiting-for-api-gateways-892310a2da02)的基础上，阐述如何为Ambassador API网关创建和部署简单的基于Java的速率限制服务。

## 设置：Docker Java Shop

在我之前的教程“[使用Kubernetes和Ambassador API网关部署Java应用](https://blog.getambassador.io/deploying-java-apps-with-kubernetes-and-the-ambassador-api-gateway-c6e9d9618f1b)”中，我将开源的Ambassador API网关添加到现有的一个部署于Kubernetes的Java（Spring Boot和Dropwizard）服务中。 如果你之前不了解这个，建议你先阅读下此教程及其他相关内容来熟悉基础知识。
本文假定你熟悉如何构建基于Java的微服务并将其部署到Kubernetes，同时已经完成安装所有的必备组件（我在本文中使用[Docker for Mac Edge](https://docs.docker.com/docker-for-mac/edge-release-notes/)，并启用其内置的Kubernetes支持。若使用minikube或远程群集应该也类似）。

## 先决条件

需要在本地安装：

- Docker for Desktop：我使用edge community edition (18.04.0-ce)，内置了对本地Kubernetes集群的支持。由于Java应用对内存有一定要求，我还将Docker可用内存增加到8G。

- 编辑器选择：Atom 或者 VS code；当写Java代码时也可以使用IntelliJ。

你可以在这里获取最新版本的“Docker Java Shop”源代码：

[https://github.com/danielbryantuk/oreilly-docker-java-shopping](https://github.com/danielbryantuk/oreilly-docker-java-shopping)

你可以通过如下命令使用SSH克隆仓库：

```bash

$ git clone git@github.com:danielbryantuk/oreilly-docker-java-shopping.git

```

第一阶段的服务和部署架构如下图所示：

![第一阶段架构](https://ws1.sinaimg.cn/large/78a165e1gy1fsvwpjxbzuj20hi0gjdga.jpg)

从图中可以看到，Docker Java Shopping应用程序主要由三个服务组成。在先前的教程中，你已经添加Ambassador API网关作为系统的“入口”。需要注意的是，Ambassador API网关直接使用Web 80号端口，因此需要确保本地运行的其他应用没有占用该端口。

## Ambassador API网关速率限制入门

我在本教程的仓库中增加了一个新文件夹 “[kubernetes-ambassador-ratelimit](https://github.com/danielbryantuk/oreilly-docker-java-shopping/tree/master/kubernetes-ambassador-ratelimit)”，用于包含Kubernetes相关配置。请通过命令行导航到此目录。此目录应包含如下文件：

```bash

(master *) oreilly-docker-java-shopping $ cd kubernetes-ambassador-ratelimit/
(master *) kubernetes-ambassador-ratelimit $ ll
total 48
0 drwxr-xr-x 8 danielbryant staff 256 23 Apr 09:27 .
0 drwxr-xr-x 19 danielbryant staff 608 23 Apr 09:27 ..
8 -rw-r — r — 1 danielbryant staff 2033 23 Apr 09:27 ambassador-no-rbac.yaml
8 -rw-r — r — 1 danielbryant staff 698 23 Apr 10:30 ambassador-rate-limiter.yaml
8 -rw-r — r — 1 danielbryant staff 476 23 Apr 10:30 ambassador-service.yaml
8 -rw-r — r — 1 danielbryant staff 711 23 Apr 09:27 productcatalogue-service.yaml
8 -rw-r — r — 1 danielbryant staff 659 23 Apr 10:02 shopfront-service.yaml
8 -rw-r — r — 1 danielbryant staff 678 23 Apr 09:27 stockmanager-service.yaml

```

你可以使用以下命令来提交Kubernetes配置：

```bash

$ kubectl apply -f .

```

通过以上命令将进行部署，这与之前架构的区别在于添加了“限速”服务。 这个服务是用Java编写的，且没有使用微服务框架。它发布了一个gRPC端点，可供Ambassador来实现限制速率。这种方案允许灵活定制你实现的速率限制算法（这点的好处，请查看我[以前的文章](https://blog.getambassador.io/rate-limiting-for-api-gateways-892310a2da02)）。

![限速架构](https://ws1.sinaimg.cn/large/78a165e1gy1fsvwvs0d8kj20hi0gj74v.jpg)

## 探索部署于Kubernetes的限速器服务

与任何其他服务一样，部署到Kubernetes的限速服务也可以根据需要进行水平扩展。 以下是Kubernetes配置文件ambassador-rate-limiter.yaml的内容：

```bash

---
apiVersion: v1
kind: Service
metadata:
  name: ratelimiter
  annotations:
    getambassador.io/config: |
      ---
      apiVersion: ambassador/v0
      kind: RateLimitService
      name: ratelimiter_svc
      service: "ratelimiter:50051"
  labels:
    app: ratelimiter
spec:
  type: ClusterIP
  selector:
    app: ratelimiter
  ports:
  - protocol: TCP
    port: 50051
    name: http
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: ratelimiter
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: ratelimiter
    spec:
      containers:
      - name: ratelimiter
        image: danielbryantuk/ratelimiter:0.3
        ports:
        - containerPort: 50051
        
```

你可以会注意到最后Docker Image处的“danielbryantuk/ratelimiter:0.3” ，但我们先需要注意，此服务在群集中运行，发布TCP端口50051。

在ambassador-service.yaml配置文件中，我还更新了Ambassador Kubernetes annotations配置，以确保通过包含“rate_limits”属性来限制对shopfront服务的请求。 我还添加了一些额外的元数据“- descriptor: Example descriptor”，这将在下一篇文章中更详细地解释。 目前我想说，这是将元数据传递到速率限制服务的好方法。

```bash

---
apiVersion: v1
kind: Service
metadata:
  labels:
    service: ambassador
  name: ambassador
  annotations:
    getambassador.io/config: |
      ---
      apiVersion: ambassador/v0
      kind:  Mapping
      name:  shopfront_stable
      prefix: /shopfront/
      service: shopfront:8010
      rate_limits:
        - descriptor: Example descriptor

```

你可以使用kubectl命令检查部署是否成功：

```bash
(master *) kubernetes-ambassador-ratelimit $ kubectl get svc
NAME               TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
ambassador         LoadBalancer   10.105.253.3     localhost     80:30051/TCP     1d
ambassador-admin   NodePort       10.107.15.225    <none>        8877:30637/TCP   1d
kubernetes         ClusterIP      10.96.0.1        <none>        443/TCP          16d
productcatalogue   ClusterIP      10.109.48.26     <none>        8020/TCP         1d
ratelimiter        ClusterIP      10.97.122.140    <none>        50051/TCP        1d
shopfront          ClusterIP      10.98.207.100    <none>        8010/TCP         1d
stockmanager       ClusterIP      10.107.208.180   <none>        8030/TCP         1d
```

6个服务看起来都不错（加上Kubernetes服务）：包含3个Java服务，2个Ambassador服务和1个ratelimiter服务。

你可以通过对shopfront服务端口的curl命令来进行测试，其应绑定在localhost外部IP的80端口上（如上文所示）：

```bash

(master *) kubernetes-ambassador-ratelimit $ curl localhost/shopfront/
<!DOCTYPE html>
<html lang="en" xmlns="http://www.w3.org/1999/xhtml">
<head>
    <meta charset="utf-8" />
...
</div>
</div>
<!-- jQuery (necessary for Bootstrap's JavaScript plugins) -->
<script src="https://ajax.googleapis.com/ajax/libs/jquery/1.12.4/jquery.min.js"></script>
<!-- Include all compiled plugins (below), or include individual files as needed -->
<script src="js/bootstrap.min.js"></script>
</body>
</html>(master *) kubernetes-ambassador-ratelimit $

```


你会注意到这生成了一些HTML，但它只是Docker Java Shop的首页，并且可以通过浏览器在[http://localhost/shopfront/](http://localhost/shopfront/)访问。但对于我们的速率限制实验，最好还是使用curl命令。

## 速率限制测试

对于这种演示性质的速率限​​制服务，我决定仅对服务本身进行限制。例如当速率限制服务计算是否需要限制请求时，唯一需要考虑的指标是在一段时间内针对特定后端的请求数量。在代码实现中使用[令牌桶算法](https://en.wikipedia.org/wiki/Token_bucket)，最多20个桶，并且每秒钟的补充10个令牌。由于速率限制与请求相关联，这意味着你可以每秒发出10次API请求且没有任何问题，并且由于存储桶最初包含20个令牌，你可以暂时超过此请求数量。但是，一旦最初额外的令牌使用完，并且仍尝试每秒发出超过10个请求，那么你将收到HTTP 429 “Too Many Requests” 状态码。这时，Ambassador API网关不再会将请求转发到后端服务。

让我看下如何通过curl发出大量请求来模拟这个操作。你需要避免显示的HTML页面（通过-output /dev/null参数）及curl请求（通过—silent参数），但需要显示不正确的HTTP响应状态（通过— show-error — fail参数）。你可以编写一个bash循环脚本并记录时间输出（以显示发出请求的时间）来创建一个非常粗颗粒度的负载发生器（可以通过CTRL-C来终止循环）：

```bash

$ while true; do curl --silent --output /dev/null --show-error --fail http://localhost/shopfront/; echo -e $(date);done
(master *) kubernetes-ambassador-ratelimit $ while true; do curl --silent --output /dev/null --show-error --fail http://localhost/shopfront/; echo -e $(date);done
Tue 24 Apr 2018 14:16:31 BST
Tue 24 Apr 2018 14:16:31 BST
Tue 24 Apr 2018 14:16:31 BST
Tue 24 Apr 2018 14:16:31 BST
...
Tue 24 Apr 2018 14:16:35 BST
curl: (22) The requested URL returned error: 429 Too Many Requests
Tue 24 Apr 2018 14:16:35 BST
curl: (22) The requested URL returned error: 429 Too Many Requests
Tue 24 Apr 2018 14:16:35 BST
Tue 24 Apr 2018 14:16:35 BST
curl: (22) The requested URL returned error: 429 Too Many Requests
Tue 24 Apr 2018 14:16:35 BST
curl: (22) The requested URL returned error: 429 Too Many Requests
Tue 24 Apr 2018 14:16:35 BST
^C

```

正你所见，从输出日志来看，前几个请求显示日期且没有错误，请求正常。过不了多久，当我的Mac上的请求循环超过每秒10次，HTTP 429错误便开始出现。
顺便说一下，我通常使用[ Apache Benchmarking “ab”](https://httpd.apache.org/docs/2.4/programs/ab.html) 负载生成工具进行这种简单的实验，但这工具在调用本地localhost会有问题（或者Docker配置也给我带来了一些问题）。

## 检验速率限制器服务

Ambassador Java限速服务的源代码在我GitHub帐户的仓库[ambassador-java-rate-limiter](https://github.com/danielbryantuk/ambassador-java-rate-limiter)中。其中也包含用于构建我推送到DockerHub容器镜像的[Dockerfile](https://github.com/danielbryantuk/ambassador-java-rate-limiter/blob/master/Dockerfile)。 以此Dockerfile作为模板，你可以对代码进行修改，然后构建并推送自己的镜像至DockerHub。你也可以修改在Docker Java Shopping仓库中的[ambassador-rate-limiter.yaml](https://github.com/danielbryantuk/oreilly-docker-java-shopping/blob/master/kubernetes-ambassador-ratelimit/ambassador-rate-limiter.yaml)文件来扩展使用你自己的速率限制服务。

## 研究Java代码

如果你深入研究Java代码，最需要关注的类应该是[RateLimiterServer](https://github.com/danielbryantuk/ambassador-java-rate-limiter/blob/master/src/main/java/io/datawire/ambassador/ratelimiter/simpleimpl/RateLimitServer.java)，它实现了在Ambassador API中使用的[Envoy代理](https://www.datawire.io/envoyproxy/)所定义的速率限制gRPC接口。我创建了一个[ratelimit.proto](https://github.com/danielbryantuk/ambassador-java-rate-limiter/blob/master/src/main/proto/ratelimit.proto)接口的一个副本，其通过Maven [pom.xml](https://github.com/danielbryantuk/ambassador-java-rate-limiter/blob/master/pom.xml)中定义的gRPC Java构建工具来使用。代码主要涉及三点：实现gRPC接口，运行gRPC服务器，并实现速率限制。下面让我们来看看。

## 实现速率限制gRPC接口

查看RateLimitServer中的内部类“RateLimiterImpl”，其对RateLimitServiceGrpc.RateLimitServiceImplBase进行扩展，你可以看到我已经重写了此抽象类中的一个方法：

```bash

public void shouldRateLimit(Ratelimit.RateLimitRequest rateLimitRequest, StreamObserver<Ratelimit.RateLimitResponse> responseStreamObserver)

```

这里使用的很多命名约定来自于Java gRPC库，进一步信息请参阅[gRPC Java文档](https://grpc.io/docs/tutorials/basic/java.html)。 话虽如此，如果查看[ratelimit.proto](https://github.com/danielbryantuk/ambassador-java-rate-limiter/blob/master/src/main/proto/ratelimit.proto)文件，你可以清楚地看到许多命名的根目录，该文件定义了在Ambassador中使用的Envoy代理所需要的速率限制接口。 例如，你可以看到此文件中定义的核心服务名为RateLimitService（第9行），并且在服务“rpc ShouldRateLimit (RateLimitRequest) returns (RateLimitResponse) {}”（第11行）中定义了一个RPC方法， 它通过上面所定义的“shouldRateLimit”方法在Java中实现。

如果有兴趣，可以看看哪些大量的由“protobuf-maven-plugin”（[pom.xml](https://github.com/danielbryantuk/ambassador-java-rate-limiter/blob/master/pom.xml)的第99行）生成的Java gRPC代码。

## 运行gRPC服务器

一旦你实现了用ratelimit.proto定义的gRPC接口，下一件事情就是创建一个gRPC服务器用来监听和回复请求。 可以根据main方法调用链来查看[RateLimitServer](https://github.com/danielbryantuk/ambassador-java-rate-limiter/blob/master/src/main/java/io/datawire/ambassador/ratelimiter/simpleimpl/RateLimitServer.java)的内容。 简而言之，main方法创建一个RateLimitServer类的实例，调用start（）方法，再调用blockUntilShutdown（）方法。 这将启动一个实例，并在指定的网络端口上发布gRPC接口，同时侦听请求。

## 实现Java速率限制


负责速率限制过程的实际Java代码包含在RateLimiterImpl内部类的shouldRateLimit（）方法（第75行）中。我没有自己实现算法，而是使用基于[令牌桶算法](https://en.wikipedia.org/wiki/Token_bucket)的Java速度限制开源库[bucket4j](https://github.com/vladimir-bukhtoyarov/bucket4j)。由于我限制了对每个服务的请求，因此每个存储桶与服务名称所绑定。对每个服务的请求都会从其所关联的存储桶中删除一个令牌。在本案例中，桶没有存储在外部数据库，而是存储在内存中的ConcurrentHashMap中。如果在生产环境中，通常会使用类似Redis的外部持久化存储方案来实现横向扩展。这里必须注意，如果在不更改每个服务桶限制的前提下水平扩展速率限制服务，那么将直接增加允许（非速率限制）请求的数量，而不是增加的服务数量。

创建bucket4j存储桶的RateLimiterImpl大致代码如下：

```bash

private Bucket createNewBucket() {
    long overdraft = 20;
    Refill refill = Refill.smooth(10, Duration.ofSeconds(1));
    Bandwidth limit = Bandwidth.classic(overdraft, refill);
    return Bucket4j.builder().addLimit(limit).build();
}

```

在下面可以看到shouldRateLimit方法的代码，它只是简单地尝试执行tryConsume（1）来尝试并使用桶中一个令牌，并返回适当的HTTP响应。

```bash

@Override
public void shouldRateLimit(Ratelimit.RateLimitRequest rateLimitRequest, StreamObserver<Ratelimit.RateLimitResponse> responseStreamObserver) {
    logDebug(rateLimitRequest);
    String destServiceName = extractDestServiceNameFrom(rateLimitRequest);
    Bucket bucket = getServiceBucketFor(destServiceName);
Ratelimit.RateLimitResponse.Code code;
    if (bucket.tryConsume(1)) {
        code = Ratelimit.RateLimitResponse.Code.OK;
    } else {
        code = Ratelimit.RateLimitResponse.Code.OVER_LIMIT;
    }
Ratelimit.RateLimitResponse rateLimitResponse = generateRateLimitResponse(code);
    responseStreamObserver.onNext(rateLimitResponse);
    responseStreamObserver.onCompleted();
}

```

代码比较容易解释。如果当前请求不需要进行速率限制，则此方法返回Ratelimit.RateLimitResponse.Code.OK; 如果当前请求由于速度限制而被拒绝，则此方法返回Ratelimit.RateLimitResponse.Code.OVER_LIMIT。根据此gRPC服务的响应，Ambassador API网关将请求传递给后端服务，或者中断请求并返回HTTP状态码429“Too Many Requests” 而不再调用后端服务。

这个简单案例可以防止一种服务过载，但也希望这能够阐明速率限制的核心概念，并且可以相对容易实现基于请求元数据（例如用户ID等）的速率限制。

## 下一阶段

本文演示了如何在Java中创建速率限制服务，并轻松与Ambassador网关所集成，同时也可以基于任何自定义的速率限制算法实现。 在本系列的最后一篇文章中，您将更深入地了解Envoy速率限制API，以便进一步学习如何设计速率限制服务。

如果有任何疑问，欢迎在Ambassador Gitter或通过[@danielbryantuk](https://twitter.com/danielbryantuk/)及[@datawireio](https://twitter.com/datawireio?lang=en)联系。
