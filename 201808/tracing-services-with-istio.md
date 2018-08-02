## istio 跟踪服务

原文链接：https://hackernoon.com/tracing-services-with-istio-e51d249da60c
发布时间：2018-07-25
译文链接：https://maiyang.me/post/2018-08-03-tracing-services-with-istio/
作者：Jeronimo (Jerry) Garcia
译者：maiyang
类型：【博客】

----

超快速发布，当 istio 注入 envoy 容器 sidecar 到你的 pod 中，每一个进出的请求都会附加一些 http 头信息，然后将他们用于跟踪。

这是 istio 所拥有的 “sidecar 注入” 方法的众多好处之一，有点干扰，但到目前为止似乎工作得很好。

好了，你可以通过以下脚本来快速的部署 jaeger 和 zipkin ：

[helm istio values](https://github.com/istio/istio/blob/master/install/kubernetes/helm/istio/values.yaml#L415)

如果你还没有启用它，然后如果你能在图表上看到一小块，你会发现如下参考：

![1_IeIAfZClvqJHvDkXTulDrg](https://ws4.sinaimg.cn/large/006tKfTcgy1ftw4urdnj6j30m10680sm.jpg)

它是 Mixer，不需要太多的深入，你可以看到 mixer 如何将统计数据传递给 zipkin，并记住 mixer 看到的一切。

因此，我们可以使用 port-forward（端口转发） 查看 jaeger 的监听：

```shell
$ kubectl port-forward -n istio-system istio-tracing-754cdfd695-ngssw
16686:16686
```

我们点击 [http://localhost:16686](http://localhost:16686) ，我们会找到 jaeger：

![1_5KEKom5j8tyagFVdSIGWlw](https://ws4.sinaimg.cn/large/006tKfTcgy1ftw4vl7uvjj30wx0kita2.jpg)

这对于跟踪和获得可能需要花费太长时间才能处理的服务的一般概念非常有趣，我强制出现错误，它看起来像：

![1_gECzUb6Hh5QjxK0-ueYT8g](https://ws4.sinaimg.cn/large/006tKfTcgy1ftw4wduq16j318g0meq5w.jpg)

如果 `nginx pod` 会调用额外的服务，那么它们也应该在那里显示，因此请记住所有入口/出口流量都是由你的 pod 中的 `envoy sidecar` 捕获的。
