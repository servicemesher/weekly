---
original: https://hackernoon.com/distributed-tracing-with-envoy-service-mesh-jaeger-c365b6191592
translator: malphi
reviewer: rootsongjc
title: "使用Envoy服务网格和Jaeger实现分布式追踪"
description: "本文用实例讲解了如何利用Envoy将gRPC转码为HTTP/JSON"
categories: "translation"
tags: ["Envoy","Jaeger"]
date: 2018-11-19
---

# 使用Envoy和Jaeger实现分布式追踪

如果你还是服务网格和Envoy的新手，我[这里](https://medium.com/@dnivra26/service-mesh-with-envoy-101-e6b2131ee30b)有一篇文章解释它们。

在微服务架构中，可观测性变得越加重要。我认为这是选择微服务这条路的必要条件之一。我的一位前同事列出了一份非常棒的[需求清单](https://news.ycombinator.com/item?id=12509533)，如果你想做微服务，你需要满足提到的这些要求。

可观测性有许多事要做：

- 监控
- 报警
- 日志集中化
- 分布式追踪

本文只讨论Envoy下的分布式跟踪，我尽量给出一个全貌来描述分布式跟踪、OpenTracing、Envoy和Jaeger是如何整合在一起工作的。在[下一篇文章](https://medium.com/@dnivra26/microservices-monitoring-with-envoy-service-mesh-prometheus-grafana-a1c26a8595fc)中，我们将讨论使用Envoy、prometheus和grafana做监控。

## 分布式追踪

随着大量的服务和请求的流转，你需要能够快速发现哪里出了问题。分布式跟踪最早由[谷歌的Dapper](https://ai.google/research/pubs/pub36356)普及开来，本质上具有在微服务的整个生命周期中跟踪请求的能力。

最简单的实现方法是在前端代理生成一个唯一的请求id（x-request-id），并将该请求id传递给与其交互的所有服务。基本上可以向所有的日志追加这一请求id。因此，如果你在kibana这样的系统中搜索唯一id，你会看到针对该特定请求的所有相关服务的日志。

这非常有用，但是它不能告诉你每个服务中请求完成的顺序，是否是并行完成的或者花费的时间。

让我们看看OpenTracing和Envoy如何帮助我们。

## OpenTracing

与其只传递一个id (x-request-id)，不如传递更多的数据，比如哪个服务位于请求的根级别，哪个服务是哪个服务的子服务等等……这可以帮我们找出所有的答案。标准的做法是使用OpenTracing。它是分布式追踪的规范，和语言无关。你可以在[这里](https://opentracing.io/speciation/)阅读更多关于规范的信息。

## Envoy

服务网格就像微服务的通信层。服务之间的所有通信都是通过网格进行的。它可以实现负载平衡、服务发现、流量转移、速率限制、指标（metrics）收集等。Envoy就是这样的一个服务网格。在我们的例子中，envoy将帮助我们生成唯一根请求id （x-request-id），生成子请求id，并将它们发送到[Jaeger](https://www.jaegertracing.io/)或[Zipkin](https://zipkin.io/)这样的追踪系统，这些系统存储、聚合追踪数据并为其提供可视化的能力。

这篇文章中我们会使用Jaeger作为追踪系统。Envoy用来生成基于zipkin或lighstep格式的追踪数据。我们会使用zipkin的标准来兼容Jaeger。

## 只要给我看代码就好...

下面的图展示了我们尝试构建的系统全貌：



![img](https://ws1.sinaimg.cn/large/006tNbRwly1fxvygc19pfj30o70a6glq.jpg)

服务安装

我们将在安装中使用docker-compose。你需要向Envoy提供配置文件。我不打算解释如何配置Envoy，我们将集中讨论与追踪相关的部分。你可以在[这里](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/v2_overview)找到更多关于配置Envoy的信息。

## 前端Envoy

前端Envoy的作用是生成根请求id，你可以通过配置去实现。下面是针对它的配置文件：
```yaml
---
tracing:
  http:
    name: envoy.zipkin
    config:
      collector_cluster: jaeger
      collector_endpoint: "/api/v1/spans"
admin:
  access_log_path: "/tmp/admin_access.log"
  address: 
    socket_address: 
      address: "127.0.0.1"
      port_value: 9901
static_resources: 
  listeners:
    - 
      name: "http_listener"
      address: 
        socket_address: 
          address: "0.0.0.0"
          port_value: 80
      filter_chains:
          filters: 
            - 
              name: "envoy.http_connection_manager"
              config:
                tracing:
                  operation_name: egress
                use_remote_address: true
                add_user_agent: true
                access_log:
                - name: envoy.file_access_log
                  config:
                    path: /dev/stdout
                    format: "[ACCESS_LOG][%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" \"%DOWNSTREAM_REMOTE_ADDRESS_WITHOUT_PORT%\"\n"
                stat_prefix: "ingress_443"
                codec_type: "AUTO"
                generate_request_id: true
                route_config: 
                  name: "local_route"
                  virtual_hosts: 
                    - 
                      name: "http-route"
                      domains: 
                        - "*"
                      routes: 
                        - 
                          match: 
                            prefix: "/"
                          route:
                            cluster: "service_a"
                http_filters:
                  - 
                    name: "envoy.router"
  clusters:
    - 
      name: "service_a"
      connect_timeout: "0.25s"
      type: "strict_dns"
      lb_policy: "ROUND_ROBIN"
      hosts:
        - 
          socket_address: 
            address: "service_a_envoy"
            port_value: 8786
    - name: jaeger
      connect_timeout: 0.25s
      type: strict_dns
      lb_policy: round_robin
      hosts:
      - socket_address:
          address: jaeger
          port_value: 9411
```

第1-8行启用追踪并配置追踪系统和它所在的位置。

第27-28行指定流量进出的位置。

第38行指出Envoy必须生成根请求id。

第66-73行配置Jaeger追踪系统。

启用追踪和配置Jaeger地址将出现在所有Envoy的配置中（前端，服务a，b和c）

## Service A

In our setup Service A is going to call Service B and Service C. The very important thing about distributed tracing is, even though Envoy supports and helps you with distributed tracing, **it is upto the services to forward the generated headers to outgoing requests**. So our Service A will forward the request tracing headers while calling Service B and Service C. Service A is a simple go service with just one end point that calls Service B and Service C internally. These are the headers that we need to pass



<iframe width="700" height="250" data-src="/media/a0fc9da7369f41612b9ff4b3dc0b3c59?postId=c365b6191592" data-media-id="a0fc9da7369f41612b9ff4b3dc0b3c59" data-thumbnail="https://i.embed.ly/1/image?url=https%3A%2F%2Favatars3.githubusercontent.com%2Fu%2F2501626%3Fs%3D400%26v%3D4&amp;key=a19fcc184b9711e1b4764040d3dc5c07" class="progressiveMedia-iframe js-progressiveMedia-iframe" allowfullscreen="" frameborder="0" src="https://hackernoon.com/media/a0fc9da7369f41612b9ff4b3dc0b3c59?postId=c365b6191592" style="display: block; position: absolute; margin: auto; max-width: 100%; box-sizing: border-box; transform: translateZ(0px); top: 0px; left: 0px; width: 700px; height: 373px;"></iframe>

Forward request tracing headers

You might wonder why the url is service_a_envoy while calling Service B. If you remember we already discussed that all the communication between the services will need to go through envoy proxy. So similarly you can pass the headers while calling Service C.

## Service B & Service C

The remaining two services need not specifically do any changes in the code since they are at the leaf level. In case these two service are going to call some other endpoint then you will have to forward the request tracing headers. And no special configurations for Envoy as well. Service B & C would look like this



<iframe width="700" height="250" data-src="/media/60f3684fa194e7663e822a21fabed994?postId=c365b6191592" data-media-id="60f3684fa194e7663e822a21fabed994" data-thumbnail="https://i.embed.ly/1/image?url=https%3A%2F%2Favatars3.githubusercontent.com%2Fu%2F2501626%3Fs%3D400%26v%3D4&amp;key=a19fcc184b9711e1b4764040d3dc5c07" class="progressiveMedia-iframe js-progressiveMedia-iframe" allowfullscreen="" frameborder="0" src="https://hackernoon.com/media/60f3684fa194e7663e822a21fabed994?postId=c365b6191592" style="display: block; position: absolute; margin: auto; max-width: 100%; box-sizing: border-box; transform: translateZ(0px); top: 0px; left: 0px; width: 700px; height: 438.984px;"></iframe>

Service B & C

So with all of this done, if you run docker-compose up and hit the Front Envoy endpoint, trace information would have been generated and pushed to Jaeger. Jaeger has a very nice to UI to visualise the traces and the trace for our setup will look like this



![img](https://cdn-images-1.medium.com/max/2000/1*nxmYEIzy8hgoGRNbJDZ5MQ.png)

trace from Jaeger

As you can see it provides the overall time taken, time taken in each part of the system, which service is calling which service, service to service relationship (service b and service c are siblings). Ill leave it to you to explore Jaeger.

You can find all the envoy configurations, code and docker compose file [here](https://github.com/dnivra26/envoy_distributed_tracing)

[**dnivra26/envoy_distributed_tracing**
*Demo for distributed tracing with envoy, zipkin|jaeger & open tracing - dnivra26/envoy_distributed_tracing*github.com](https://github.com/dnivra26/envoy_distributed_tracing)

Thats it folks. Thanks. Please let me know your feedback.

If you are looking for an Envoy xDS server, my colleague has built [one](https://github.com/tak2siva/Envoy-Pilot) . Do check it out.

[Here](https://medium.com/@dnivra26/microservices-monitoring-with-envoy-service-mesh-prometheus-grafana-a1c26a8595fc) is the next article(Monitoring with Envoy, Prometheus & Grafana) in series.