
<center>Kong mesh深度分析报告</center>
==

Kong是一个基于OpenResty (Nginx) 封装的微服务中间件产品，在微服务架构体系中，作为API网关以及API中间件（kubernetes ingress）提供服务。由于其天生具备Nginx的高性能、nginx-lua插件的可定制性，再加上完善的社区以及齐全的文档，在中小企业用户群非常受欢迎，拥有较好的群众基础。
2018年8月，kong发布了1.0 GA版本，正式宣布其支持service mesh，并提供社区版以及企业版2个版本。下面我们从Demo、配置、功能这3方面，对kong mesh进行体验及分析。

## Demo体验

Kong社区提供了kong mesh的demo (<span>https://github.com/Kong/kong-mesh-dist-kubernetes</span>），该demo是实现的是tcp四层透明代理转发业务。
该demo主要做的事情是：提供两个服务servicea以及serviceb，serviceb作为服务端，通过ncat监听8080端口，接受外部的TCP消息；servicea作为client端，
通过ncat将当前server的时间发往serviceb。Demo的运行效果如下：<br/>

客户端，每隔两秒会发送一次时间戳到服务端
