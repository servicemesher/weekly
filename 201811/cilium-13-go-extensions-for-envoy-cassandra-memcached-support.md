October 23, 2018

# Cilium 1.3：支持Envoy，Cassandra和Memcached的Go语言扩展

![Cilium Kubernetes](https://ws3.sinaimg.cn/large/006tNbRwly1fwqjul334zj30je07p0t1.jpg)

我们很高兴地宣布Cilium 1.3发布了。这个版本加入了几个新特性。主要的亮点是为[Envoy](https://github.com/envoyproxy/envoy)添加了Go语言扩展，同样，Cassandra以及带有策略执行能力的Memcached协议解析器，也都实现了Go语言的扩展。

像往常一样，向Cilium开发人员的整个社区发出了巨大的呼声，他们在1.2到1.3年间提交了785个提交。

## 什么是Envoy的Go语言扩展？

从1.0版本开始，我们一直依赖[Envoy](https://github.com/envoyproxy/envoy)处理所有的HTTP、gRPC以及HTTP的派生如Elasticsearch。随着社区讨论如何扩大支持7层协议的范围，Envoy作为推动未来协议补充的首选平台是显而易见的。焦点迅速转移到寻找简化Envoy可扩展性的方法，并且允许重用现有的开源项目，如CNCF项目[Vitess](https://vitess.io/)。于是实现Envoy的Go扩展的想法就诞生了。

在Cilium 1.3中，我们引入了Envoy的Go扩展作为其Beta特性。

![Envoy Golang Extension Architecture](https://ws4.sinaimg.cn/large/006tNbRwly1fwql07xp02j30lp0buaay.jpg)

- **扩展的透明注入：**在Cilium的帮助下，连接被透明地重定向到Envoy，而不需要修改应用程序或pod。重定向基于目标端口配置，可以根据labels、IPs、DNS以及ingress和egress连接的服务名称限定到源或目标服务，并通过扩展的名称将连接映射到扩展。重定向是通过CiliumNetworkPolicy CRD或REST API配置的。Envoy可以被配置为在每个pod中作为sidecar或作为每个node的独立代理运行。
- **完全分布式：**Go扩展完全分布在每个Envoy节点或pod内部，不需要集中化的控制平面进行数据处理。当然，go扩展本身可以调用任意的外部控制面板组件来报告遥测数据或验证请求。
- **动态扩展映射：**Go扩展被设计为共享库提供给Envoy。Cilium配置Envoy可以根据配置的重定向自动加载相应的Go扩展，并在连接数据时调用它。未来的版本将支持在运行时更新和重新加载扩展，而无需重启Envoy和不丢失连接状态。
- **通过CRD配置扩展：**通过CRD或REST API使用通用键值对配置Go扩展。这允许传递配置如安全策略、安全令牌或其他配置，而无需让Envoy知道它。
- **通用访问日志：**与配置类似，扩展可以返回通用键值对，这些键值对将提取的可见性传递到访问日志层。
- **沙盒化：**沙盒确保任何解析器的不稳定性都不能破坏Envoy的成熟核心。受Matt Klein发表的文章[Exceptional Go](https://medium.com/@mattklein123/exceptional-go-1dd1488a6a47)启发，解析器被容许panic或抛出异常。当panic发生时，信息被记录到访问日志中，TCP连接与被关闭的请求关联。

## Cilium是什么？

Cilium是一种开源软件，可以透明地提供和保护使用诸如Kubernetes、Docker和Mesos等Linux容器管理平台部署的应用程服务之间的网络和API连接。

Cilium的基础是一种新的Linux内核技术BPF，它支持在Linux内部动态的注入安全、可见性和网络控制逻辑。除了提供传统的网络层安全，BPF的灵活性还能提供API和流程级别的安全，保护容器或pod间通信。因为BPF在Linux内核中运行，Cilium的安全策略可以在不修改程序代码或容器配置的情况下应用和更新。

有关Cilium更详细的介绍请参见**[Introduction to Cilium](https://cilium.readthedocs.io/en/v1.3/intro/)**

## Envoy是什么？

Envoy is an L7 proxy and communication bus designed for large modern service-oriented architectures. The project was born out of the belief that:

Envoy是一个7层代理和通信总线，被设计用于大型的面向服务的架构。这个项目诞生于以下理念：

> 网络应该对应用程序透明。当网络和应用程序出现问题时，应该很容易确定问题的根源。

你可以通过Envoy的文档 [What is Envoy](https://www.envoyproxy.io/docs/envoy/latest/intro/what_is_envoy)了解更多关于Envoy的内容。

# How to write an Envoy Go extension

Writing extensions for Envoy is simple. To illustrate this, we will implement a basic protocol parser for the R2-D2 control protocol and implement filtering logic to exclude any control request that contains the string "C-3PO".

[![Envoy Golang Extension Architecture](https://cilium.io/static/envoy_go-4e6a76dec0afb23a09dd57bbe7f08975-84ad3.png)](https://cilium.io/static/envoy_go-4e6a76dec0afb23a09dd57bbe7f08975-91d6d.png)

The primary API for an extension to implement is the `OnData()` function which is invoked whenever Envoy is receiving data on a connection that has been mapped to an extension via the `CiliumNetworkPolicy`. The function must parse the data and return one of the following verdicts:

- **MORE:** Parser needs more *n* more bytes to continue parsing.
- **PASS:** Pass along *n* bytes of the data stream.
- **DROP:** Drop *n* bytes of the data stream.
- **INJECT:** Inject *n* bytes of data in the specified direction.
- **ERROR:** A parsing error has occurred, the connection must be closed.
- **NOP:** Do nothing.

In order to register the extension, a parser factory is created which must implement a `Create()`function. The function is called whenever Envoy has established a new connection for which the parser should be used.

```go
import (
        "github.com/cilium/cilium/proxylib/proxylib"
)

type parser struct{
        connection *proxylib.Connection
}

func (p *parser) OnData(reply, endStream bool, dataArray [][]byte) (proxylib.OpType, int) {
        data := string(bytes.Join(dataArray, []byte{}))
        msgLen := strings.Index(data, "\r\n")
        if msgLen < 0 {
                return proxylib.MORE, 1 // No delimiter, request more data
        }

        msgStr := data[:msgLen]
        msgLen += 2 // Inlcude the "\r\n" in the request

        if reply {
                return proxylib.PASS, msgLen // Pass responses without additional parsing
        }

        if strings.Contains(msgStr, "C-3PO") {
                return proxylib.DROP, msgLen
        }

        return proxylib.PASS, msgLen
}

type factory struct{}

func (f *factory) Create(connection *proxylib.Connection) proxylib.Parser {
        return &parser{connection: connection}
}

func init() {
        proxylib.RegisterParserFactory("r2d2", &factory{})
}
```

Finally, hook the new parser into the proxylib by importing the new parser package into the proxylib package. This will include the parser in the `libcilium.so` that is loaded by Envoy. Edit `proxylib/proxylib.go`:

```go
import (
        [...]
        _ "github.com/cilium/cilium/proxylib/r2d2"
)
```

The above example leaves out the configuration of the extension, integration into the policy repository and all aspects of access logging. See the guide [Envoy Go Extensions](https://cilium.readthedocs.io/en/v1.3/envoy/extensions/) for a step by step guide on how to write a Go extension.

# Cassandra Support (Beta)

[![Cassandra Logo](https://cilium.io/static/cassandra_go2-04756b327d0e0b7bd9c44efa00e2839a-717ae.png)](https://cilium.io/static/cassandra_go2-04756b327d0e0b7bd9c44efa00e2839a-717ae.png)

[Cassandra](https://github.com/apache/cassandra) is a popular NoSQL database management system. It is often operated at large scale and accessed by many services and often shared between teams. Cilium 1.3 introduces protocol support for the Apache [Cassandra](https://github.com/apache/cassandra) protocol provide visibility and policy enforcement.

The Cassandra Go extension is capable to provide visibility and enforcement on the following protocol fields:

- **query_action:** The action performed on the database `SELECT`, `INSERT`, `UPDATE`, ... The field is always matched as an exact match.
- **query_table:** The table on which the query is executed on. Matching is possible with a regular expression.

### Example: How the Empire restricts Cassandra access by table

The following example shows how the Empire is exposing limited access to the Empire's Cassandra cluster running on port 9042 to outposts. Outposts are identified by the label `app=empire-outpost`and have the following privileges:

- `SELECT` access on the tables "system.*` and "system_schema.*"
- `INSERT` on the table "attendance.daily_records". Note that the outposts can't read from the tables and thus can't read the daily records from other outposts.

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
description: "Allow only permitted requests to empire Cassandra server"
metadata:
name: "secure-empire-cassandra"
specs:
- endpointSelector:
  matchLabels:
    app: cass-server
ingress:
- fromEndpoints:
  - matchLabels:
      app: empire-outpost
  toPorts:
  - ports:
    - port: "9042"
      protocol: TCP
    rules:
      l7proto: cassandra
      l7:
      - query_action: "select"
        query_table: "system\\..*"
      - query_action: "select"
        query_table: "system_schema\\..*"
      - query_action: "insert"
        query_table: "attendance.daily_records"
```

This is a simple example, see the [Cassandra getting started guide ](https://cilium.readthedocs.io/en/v1.3/gettingstarted/cassandra/)for more complex examples.

# Memcached Support (Beta)

[![Memcached Logo](https://cilium.io/static/memcached-d375d3fed73aa5efdb8e317b9337d1e8-e7d6c.png)](https://cilium.io/static/memcached-d375d3fed73aa5efdb8e317b9337d1e8-e7d6c.png)

Memcached is a popular distributed in-memory key-value store that is often used for caching purposes or to share small chunks of arbitrary data between services. With the addition of a memcached parser golang extension to Envoy, Cilium can now enforce security rules to restrict memcached clients to certain commands such as read or write but also to certain key prefixes.

## Example: How the Rebels secure a shared memcached service

In the following example, the Rebels have started running a memcached service identified by the label `app=memcached`. Several services are interacting with the memcached services and different rules are being applied:

- The fleet maintenance service identified by `function=fleet-maintenance` is granted read and write access to all keys with the prefix `alliance/fleet`. Access to any other key is prohibited.
- The fleet monitoring service identified by `function=fleet-monitoring` is only granted read access on keys with the prefix `alliance/fleet`. Write access to keys in the prefix or access to any key outside of the prefix is prohibited.
- All Jedis identified by the label `role=jedi` have full to the entire Memcached service and can access all keys.

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
description: "Secure the Rebel memcached service"
metadata:
  name: "secure-rebel-alliance-memcache"
specs:
  - endpointSelector:
      matchLabels:
        app: memcached
    ingress:
    - fromEndpoints:
      - matchLabels:
          function: fleet-maintanence
      toPorts:
      - ports:
        - port: "11211"
          protocol: TCP
        rules:
          l7proto: memcache
          l7:
          - command: "writeGroup"
            keyPrefix: "alliance/fleet/"
          - command: "get"
            keyPrefix: "alliance/fleet/"
    - fromEndpoints:
      - matchLabels:
          function: fleet-monitoring
      toPorts:
      - ports:
        - port: "11211"
          protocol: TCP
        rules:
          l7proto: memcache
          l7:
          - command: "get"
            keyPrefix: "alliance/fleet/"
    - fromEndpoints:
      - matchLabels:
          role: jedi
      toPorts:
      - ports:
        - port: "11211"
          protocol: TCP
        rules:
          l7proto: memcache
          l7:
          - command:
```

For a full example using Memcached, see the [Memcached getting started guide](https://cilium.readthedocs.io/en/v1.3/gettingstarted/memcached/).

# Community

## Linux Foundation Core Infrastructure Initiative Best Practices

[![CII Best Practices](https://bestpractices.coreinfrastructure.org/projects/1269/badge)](https://bestpractices.coreinfrastructure.org/projects/1269)

We are committed to developing Cilium in the open and with best open source best practices. This includes a strong commitment to security. We are happy to announce that we have completed all work to meet the [CII Best Practices](https://bestpractices.coreinfrastructure.org/en) of the Linux Foundation [Core Infrastructure Initiative](https://www.coreinfrastructure.org/). You can learn more about the badge program [here](https://www.coreinfrastructure.org/programs/badge-program/).

## Introduction of Special Interest Groups (SIGs)

The community is growing and getting more diverse in interest. In order to ensure the scalability of the project, we are officially introducing special interest groups (SIGs) to help provide some structure. The following SIGs have been created already:

- **Datapath:** (#sig-datapath) Owner of all BPF and Linux kernel related datapath code.
- **Documentation:** (#sig-docs) All documentation related discussions
- **Envoy:** (#sig-envoy) Envoy, Istio and maintenance of all L7 protocol parsers.
- **Policy:** (#sig-policy) All topics related to policy. The SIG is responsible for all security relevant APIs and the enforcement logic.
- **Release Management:** (#launchpad) Responsible for the release management and backport process.

Anyone can propose additional SIGs. The process is simple and documented [here](https://cilium.readthedocs.io/en/v1.3/community/#how-to-create-a-sig)

# 1.3 Release Highlights

- **Go extensions for Envoy**
  - Exciting new extension API for Envoy using Go including a generic configuration and access logging API. (Beta)
- **Cassandra & Memcached protocol support**
  - New protocol parsers for Cassandra and Memcached implemented using the new Envoy Go extensions. Both parsers provide visibility and security policy enforcement on operation type and key/table names using exact matches, prefix matches, and regular expressions. (Beta)
- **Security**
  - TTLs support for DNS/FQDN policy rules
  - Introduction of well-known identities for kube-dns, coredns, and etcd-operator.
  - New security identity "unmanaged" to represent pods which are not managed by Cilium.
  - Improved security entity "cluster" which allows defining policies for all pods in a cluster (managed, unmanaged and host networking).
- **Additional Metrics & Monitoring**
  - New "cilium metrics list" command to list metrics via CLI.
  - Lots of additional metrics: connection tracking garbage collection, Kubernetes resource events, IPAM, endpoint regenerations, services, and error and warning counters.
  - New monitoring API with more efficient encoding/decoding protocol. Used by default with fallback for older clients.
- **Networking Improvements**
  - Split of connection tracking tables into TCP and non-TCP to better handle the mix of long and short-lived nature of each protocol.
  - Ability to specify the size of the connection tracking tables via ConfigMap.
  - Better masquerading behavior for traffic via NodePort and HostPort to allow pods to see the original source IP if possible.
- **Full Key-value store Resiliency**
  - Introduced ability to re-construct the kvstore contents immediately after loss of any state. Allows to restore etcd from backup or to completely wipe it for a running cluster with minimal impact. (Beta)
- **Efficiency & Scale**
  - Significant improvements in the cost of calculating policy of individual endpoints. Work continues on this subject.
  - New grace period when workloads change identity to minimize connectivity impact throughout identity change.
  - More efficient security identity allocation algorithm.
  - New generic framework to detect and ignore Kubernetes event notifications for which Cilium does not need to take action.
  - Improvements in avoiding unnecessary BPF compilations to reduce the CPU overhead caused by it. Initial work to scope BPF templating to avoid compilation altogether.
- **Kubernetes**
  - Added support for Kubernetes 1.12
  - Custom columns for the CiliumEndpoints CRD (Requires Kubernetes 1.11)
  - Removed cgo dependency from cilium-cni for compatibility with ulibc
  - Removed support for Kubernetes 1.7
- **Documentation**
  - New Ubuntu 18.04 guide
  - Coverage of latest BPF runtime features such as BTF (BPF Type Format).
  - Documentation for VM/host firewall requirements to run multi-host networking.
- **Long Term Stable (LTS) Release**
  - 1.3 has been declared an LTS release and will be supported for the next 6 months with backports.

## Upgrade Instructions

As usual, follow the [upgrade guide](https://cilium.readthedocs.io/en/v1.3/install/upgrade/#upgrading-minor-versions) to upgrade your Cilium deployment. Feel free to ping us on [Slack](http://cilium.io/slack).

## Release

- Release Notes & Binaries: [1.3.0](https://github.com/cilium/cilium/releases/tag/1.3.0)
- Container image: `docker.io/cilium/cilium:v1.3.0`