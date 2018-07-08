# The Circonus Istio Mixer Adapter

> 原文地址：<https://www.circonus.com/2017/12/circonus-istio-mixer-adapter/>
>
> 作者：Fred Moyer
>
> 译者：[陈冬](https://github.com/shaobai)


在 Circonus，我们有悠久的开源软件参与的传统。因此，当我们看到 Istio 提供了一个精心设计接口，通过适配器连接 syndicate 服务遥测，我们知道一个 Circonus 适配器将是一个自然的契合。Istio 已经被设计成提供高性能、高可扩展的应用控制平面，并且 Circonus 也被设计为具有性能和可扩展行的核型原则。

今天我们很高兴的宣布 [Istio 服务网格](https://istio.io/) 的 Circonus 适配器的可用性。这篇博客文章将介绍这个适配器的开发，并向您展示如何快速启动并运行它。我们知道你会对此非常有兴趣，因为 Kubernetes 和 Istio 提供你能力扩展到 Circonus 设计的水平，高于其他遥测解决方案。

如果你不知道什么是服务网格，你并不孤单，但希望是你已经使用很多年了。互联网的路由基础设施就是一个服务网格；它有利于 TCP 重传、访问控制、动态路由、流量规划等。占主导地位但web整体性web应用正在为微服务组成的应用让路。Istio 通过一个  [sidecar proxy](https://www.envoyproxy.io/docs/envoy/latest/) 提供基于容器的分布式应用程序的控制平面功能。它为服务的操作人员提供了丰富的功能来控制 [Kubernetes](https://kubernetes.io/) 编排的服务集合，而不需要服务本身来实现任何控制平面的功能集合。

 Istio 混合器 [Mixer](https://istio.io/docs/concepts/policies-and-telemetry/overview/) 提供了一个 [适配器](https://istio.io/blog/2017/adapter-model/) 模型，它允许我们创建用于，它允许我们通过创建用于外部基础设施后端接口的混合器的 [处理器](https://istio.io/docs/concepts/policies-and-telemetry/config/#handlers) 来开发适配器。混合器还提供了一组模版，每个模板都为适配器提供了不同的元数据集。在例如 Circonus 适配器之类的度量适配器，该元数据集包括诸如*请求持续时间*（*request duration*）、*请求计数*（*request count*）、*请求有效负载大小*（*request payload size*）等度量。要激活 Istio 启用的 Kubernetes 集群中的 Circonus 适配器，只需要使用 istioctl 命令将 Circonus  [运算配置](https://github.com/istio/istio/blob/master/mixer/adapter/circonus/operatorconfig/config.yaml)(`地址有误，待修正`) 注入到 k8s 集群中，metrics 将开始流动。

 以下是一个关于混合器如何与这些外部后端服务交互的架构视图：

![](https://ws4.sinaimg.cn/large/006tNc79gy1ft2kovaczfj319u0zftc9.jpg)

Istio 还包含了 StatsD 和 Prometheus 的 metrics 适配器。然而，Circonus 适配器与其他适配器又存在一些区别。首先，Circonus 适配器允许我们将请求持续时间作为一个直方图来收集，而不仅仅是记录固定的百分位数。这使我们能够计算任何时间窗上的任意分位数，并对所收集的直方图进行统计分析。第二，数据可以基本上永久保留。第三，telemetry 数据被保存在持久的环境中，而独立于 Kubernetes 管理的任何短暂资产之外。

让我们来看看，数据是如何从 Istio 到 Circonus中的。Istio 的适配器框架暴露了很多适合适配器开发者的方法。Istio 处理的每个请求都生成了一组度量实例用来调用 *HandleMetric()* 方法。在我们的运算配置中，我们可以指定我们要采用的度量，以及他们的类型：

```
spec:
  # HTTPTrap url, replace this with your account submission url
  submission_url: "https://trap.noit.circonus.net/module/httptrap/myuuid/mysecret"
  submission_interval: "10s"
  metrics:
  - name: requestcount.metric.istio-system
    type: COUNTER
  - name: requestduration.metric.istio-system
    type: DISTRIBUTION
  - name: requestsize.metric.istio-system
    type: GAUGE
  - name: responsesize.metric.istio-system
    type: GAUGE
```

在这里，我们配置了一个服从 [HTTPTrap](https://login.circonus.com/resources/docs/user/Data/CheckTypes/HTTPTrap.html) 检查的 URL 同时间断发送度量的 Circonus 处理程序。在这个例子中，我们指定了四个度量的集合，以及他们的类型。请注意，我们将 *请求持续时间* 对量作为一个 *DISTRIBUTION* 类型来收集，将作为 Circonus 中的直方图进行处理。这将保持时间的保真度，而不是平均该度量，或者在记录之前计算百分位数（这两种技术都失去了信号的值）。

对每个请求，对每个指定的度量请求调用 *HandleMetric() * 方法。看如下代码：

```
// HandleMetric submits metrics to Circonus via circonus-gometrics
func (h *handler) HandleMetric(ctx context.Context, insts []*metric.Instance) error {
    for _, inst := range insts {
        metricName := inst.Name
        metricType := h.metrics[metricName]

        switch metricType {
        case config.GAUGE:
            value, _ := inst.Value.(int64)
            h.cm.Gauge(metricName, value)

        case config.COUNTER:
            h.cm.Increment(metricName)

        case config.DISTRIBUTION:
            value, _ := inst.Value.(time.Duration)
            h.cm.Timing(metricName, float64(value))
        }
    }
    return nil
}
```
在这里我们可以看到，使用一个混合器的上下文以及一组 metric 实例来调用 *HandleMetric()* 方法，我们遍历每个实例，确定它的类型，并调用适当的 *circonus-gometrics* 方法。在这个框架中，metric 处理器包含一个 *circonus-gometrics* 对象，并提交实际的度量值来实现。设置处理器还是比较复杂的，但并不是最复杂的事情：
```
// Build constructs a circonus-gometrics instance and sets up the handler
func (b *builder) Build(ctx context.Context, env adapter.Env) (adapter.Handler, error) {

    bridge := &logToEnvLogger{env: env}

    cmc := &cgm.Config{
        CheckManager: checkmgr.Config{
            Check: checkmgr.CheckConfig{
                SubmissionURL: b.adpCfg.SubmissionUrl,
            },
        },
        Log:      log.New(bridge, "", 0),
        Debug:    true, // enable [DEBUG] level logging for env.Logger
        Interval: "0s", // flush via ScheduleDaemon based ticker
    }

    cm, err := cgm.NewCirconusMetrics(cmc)
    if err != nil {
        err = env.Logger().Errorf("Could not create NewCirconusMetrics: %v", err)
        return nil, err
    }

    // create a context with cancel based on the istio context
    adapterContext, adapterCancel := context.WithCancel(ctx)

    env.ScheduleDaemon(
        func() {

            ticker := time.NewTicker(b.adpCfg.SubmissionInterval)

            for {
                select {
                case <-ticker.C:
                  cm.Flush()
                case <-adapterContext.Done()
                  ticker.Stop()
                  cm.Flush()
                  return
                }
            }
          })
    metrics := make(map[string])config.Params_MetricInfo_Type)
    ac := b.adpCfg
    for _, adpMetric := range ac.Metrics {
        metrics[adpMetricName] = adpmetric.Type
    }
    return &handler{cm: cm, env: env, metrics: metrics, cancel: adapterCancel}, nil
}
```
混合器提供了一个生成器类型，我们定义了构建方法。再次，混合器的上下文被传递，以及表示混合器配置的环境变量。我们创建了一个新的 *circonus-gometrics* 对象，并故意禁用了自动 metrics 刷新。我们这样做是因为混合器要求我们在使用 *env.ScheduleDaemon()* 方法时在 panic 处理器中包装所有的 goroutines 。你会注意到我们以及通过 *context.WithCancel* 创建了自己的 *adapterContext* 。这使得我们可以通过混合器提供的 *Close()* 方法中调用 *h.cancel()* 来关闭 metrics 刷新 goroutine 。我们还希望将任何日志事件从 CGM (*circonus-gometrics*) 发送到混合气的日志中。混合器提供了一个基于 glog 的  *env.Logger()* 接口，但 CGM 使用的是标准的 Golang 日志。我们如何解决这种不匹配的阻碍？通过一个 logger bridge，任何 CGM 生成的日志记录语句都可以传递给混合器。

```
// logToEnvLogger converts CGM log package writes to env.Logger()
func (b logToEnvLogger) Write(msg []byte) (int, error) {
    if bytes.HasPrefix(msg, []byte("[ERROR]")) {
        b.env.Logger().Errorf(string(msg))
    } else if bytes.HasPrefix(msg, []byte("[WARN]")) {
        b.env.Logger().Warningf(string(msg))
    } else if bytes.HasPrefix(msg, []byte("[DEBUG]")) {
        b.env.Logger().Infof(string(msg))
    } else {
        b.env.Logger().Infof(string(msg))
    }
    return len(msg), nil
}
```

全适配器的代码，可以查看 Istio github 库 [点我](https://github.com/istio/istio/tree/master/mixer/adapter/circonus)

尽管如此，让我们看看在执行过中是什么样子的。我安装了 Google 的 Kubernetes 引擎部署，使用 Circonus 加载了 Istio 的开发版本，并部署了于 Istio 一起提供的示例 BookInfo 应用程序。下面的图像是从请求到应用程序的请求持续时间分布的热图。你会注意到高亮显示的时间片段的直方图覆盖。我添加了一个覆盖，添加了一个中位数、第九十和百分位的响应时间；
