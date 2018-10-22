---
original: https://venilnoronha.io/raw-tcp-traffic-shaping-with-istio-1.1.0
translator: malphi
reviewer: rootsongjc
title: "用Istio进行TCP流量加工"
description: "用Istio进行TCP流量加工"
categories: "译文"
tags: ["Istio","tutorial"]
date: 2018-10-22
---

# 用Istio进行TCP流量加工

[Istio](https://istio.io/)通过[虚拟服务](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#VirtualService), [目标规则](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#DestinationRule), [Gateway](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#Gateway)等概念提供了复杂的路由机制。Istio 1.0通过[加权路由定义](https://istio.io/docs/tasks/traffic-management/traffic-shifting/#apply-weight-based-routing).启用了HTTP流量转移。我通过我的pull request[on Envoy](https://github.com/envoyproxy/envoy/pull/4430) 和 [on Istio](https://github.com/istio/istio/pull/9112)为TCP/TLS服务提供类似的特性。这一特性已经在Envoy [1.8.0](https://www.envoyproxy.io/docs/envoy/latest/intro/version_history#oct-4-2018)中发布了。Istio中的特性也会在即将发布的[1.1.0](https://github.com/istio/istio/releases/)版本中提供使用。

![Istio](https://ws4.sinaimg.cn/large/006tNbRwly1fw3hlzj8krj304o06m3ya.jpg)

在本文中，我们将用[Go](https://golang.org/)编写的一个简单的TCP Echo服务，用[Docker](https://www.docker.com/)将其容器化并部署到[Kubernetes](https://kubernetes.io/)上，并通过练习Istio的加权TCP路由特性来理解其在生产服务中的行为。

## TCP Echo服务

在本文中，我们将创建一个简单的监听连接的TCP服务，并在客户端请求数据前加上一个简单的前缀，将其作为响应返回。图示如下：

![TCP Client - Server Architecture](https://ws1.sinaimg.cn/large/006tNbRwly1fwgz3b6bpoj30r607qgm6.jpg)

让我们看一下TCP Echo服务端的Go代码：

```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"net"
	"os"
)

// main作为程序入口点
func main() {
	// 通过程序入参获得端口和前缀
	port := fmt.Sprintf(":%s", os.Args[1])
	prefix := os.Args[2]

	// 在给定端口上创建tcp监听
	listener, err := net.Listen("tcp", port)
	if err != nil {
		fmt.Println("failed to create listener, err:", err)
		os.Exit(1)
	}
	fmt.Printf("listening on %s, prefix: %s\n", listener.Addr(), prefix)

	// 监听新的连接
	for {
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("failed to accept connection, err:", err)
			continue
		}

		// 启用goroutine处理连接
		go handleConnection(conn, prefix)
	}
}

// handleConnection 处理连接的生命周期
func handleConnection(conn net.Conn, prefix string) {
	defer conn.Close()
	reader := bufio.NewReader(conn)
	for {
		// 读取客户端请求数据
		bytes, err := reader.ReadBytes(byte('\n'))
		if err != nil {
			if err != io.EOF {
				fmt.Println("failed to read data, err:", err)
			}
			return
		}
		fmt.Printf("request: %s", bytes)

		// 添加前缀作为response返回
		line := fmt.Sprintf("%s %s", prefix, bytes)
		fmt.Printf("response: %s", line)
		conn.Write([]byte(line))
	}
}
```

测试这个程序，复制上面代码到`main.go`文件，并执行命令如下：

```bash
$ go run -v main.go 9000 hello
listening on [::]:9000, prefix: hello
```

我们可以通过 `nc` ([Netcat](https://en.wikipedia.org/wiki/Netcat))在TCP层面上和这段程序交互。要发送请求，我们可以使用BusyBox容器，如下所示：

```bash
$ docker run -it --rm busybox sh -c 'echo world | nc docker.for.mac.localhost 9000'
hello world
```

就像你看到的，在请求“world”前面加上了“hello”，“hello world”作为response。注意，我正在执行的BusyBox容器基于 [Docker for Mac](https://docs.docker.com/docker-for-mac/)，这就是为什么我访问Echo服务用`docker.for.mac.localhost`代替了`localhost`。

## 容器化TCP Echo服务器

因为我们最终想要在Kubernetes集群上运行TCP Echo服务器，现在让我们将它放到容器中，并发布镜像到 [Docker Hub](https://hub.docker.com/).

首先，用下面的内容创建`Dockerfile`：

```dockerfile
# 使用golang容器构建可执行文件
FROM golang:1.11 as builder
WORKDIR /go/src/github.com/venilnoronha/tcp-echo-server/
COPY main.go .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main main.go

# 负责bin文件到基于alpine的分离容器
FROM alpine:3.8
RUN apk --no-cache add ca-certificates
WORKDIR /bin/
COPY --from=builder /go/src/github.com/venilnoronha/tcp-echo-server/main .
ENTRYPOINT [ "/bin/main" ]
CMD [ "9000", "hello" ]
EXPOSE 9000
```

现在构建容器并发布镜像到Docker Hub：

```bash
$ docker build -t vnoronha/tcp-echo-server:latest .
Sending build context to Docker daemon  60.93kB
...
Successfully built d172af115e18
Successfully tagged vnoronha/tcp-echo-server:latest

$ docker push vnoronha/tcp-echo-server:latest
The push refers to repository [docker.io/vnoronha/tcp-echo-server]
b4cc76510de6: Pushed
...
latest: digest: sha256:0a45b5a0d362db6aa9154717ee3f2b... size: 949
```

## 部署TCP Echo服务到Kubernetes

### 服务配置

我们部署2个版本的TCP ECHO服务，用不同的前缀演示路由行为。创建`service.yaml`，用Kubernetes [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 和2个 [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 构建2个版本的TCP ECHO服务。

```
apiVersion: v1
kind: Service
metadata:
  name: tcp-echo-server
  labels:
    app: tcp-echo-server
    istio: ingressgateway # use istio default controller
spec:
  selector:
    app: tcp-echo-server
  ports:
  - port: 9000
    name: tcp
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: tcp-echo-server-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: tcp-echo-server
        version: v1
    spec:
      containers:
      - name: tcp-echo-server
        image: vnoronha/tcp-echo-server:latest
        args: [ "9000", "one" ] # prefix: one
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9000
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: tcp-echo-server-v2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: tcp-echo-server
        version: v2
    spec:
      containers:
      - name: tcp-echo-server
        image: vnoronha/tcp-echo-server:latest
        args: [ "9000", "two" ] # prefix: two
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9000
```

### Deploying Minikube

[Minikube](https://kubernetes.io/docs/setup/minikube/) serves as a great tool for locally developing on Kubernetes. I’ve started my Minikube instance with the following command.

```
$ minikube start --bootstrapper kubeadm       \
                 --memory=8192                \
                 --cpus=4                     \
                 --kubernetes-version=v1.10.0 \
                 --vm-driver=virtualbox
Starting local Kubernetes v1.10.0 cluster...
...
Kubectl is now configured to use the cluster.
Loading cached images from config file.
```

### Installing Istio

At the time of writing this article, Istio 1.1.0 wasn’t released. Therefore, I resorted to an Istio [Daily Pre-Release](https://gcsweb.istio.io/gcs/istio-prerelease/daily-build/master-20181017-09-15/) to demonstrate the new feature. Please refer to the [Istio Docs](https://istio.io/docs/setup/kubernetes/download-release/) to learn to download and configure Istio.

Once configured, here’s an easy way to fully deploy Istio components.

```
$ kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml
customresourcedefinition.apiextensions.k8s.io/virtualservices.networking.istio.io created
...
customresourcedefinition.apiextensions.k8s.io/templates.config.istio.io created
customresourcedefinition.apiextensions.k8s.io/handlers.config.istio.io created

$ kubectl apply -f install/kubernetes/istio-demo.yaml
namespace/istio-system created
...
destinationrule.networking.istio.io/istio-policy created
destinationrule.networking.istio.io/istio-telemetry created 
```

### Deploying TCP Echo Servers With Istio Proxies

Since we want to demonstrate Istio’s routing mechanics, let’s deploy the `tcp-echo-server` with the Istio Proxy side-car as shown below.

```
$ kubectl apply -f <(istioctl kube-inject -f service.yaml)
service/tcp-echo-server created
deployment.extensions/tcp-echo-server-v1 created
deployment.extensions/tcp-echo-server-v2 created
```

We can verify that the services are running via the following commands.

```
$ kubectl get pods
NAME                                  READY     STATUS    RESTARTS   AGE
tcp-echo-server-v1-78684f5697-sv5r5   2/2       Running   0          56s
tcp-echo-server-v2-74bf9999c8-hhhf9   2/2       Running   0          56s

$ kubectl logs tcp-echo-server-v1-78684f5697-sv5r5 tcp-echo-server
listening on [::]:9000, prefix: one

$ kubectl logs tcp-echo-server-v2-74bf9999c8-hhhf9 tcp-echo-server
listening on [::]:9000, prefix: two
```

## Weighted TCP Routing With Istio

This is the final part of this exercise where we define a `VirtualService`,`DestinationRule` and a `Gateway` with weighted routes and verify the system behavior.

### Routing Configuration

We create a `DestinationRule` with 2 `subsets` to represent the 2 versions of the TCP Echo Server. The `Gateway` allows traffic to access the services on TCP port `9000`. Finally, the `VirtualService` specifies that 80% of the traffic must be routed to TCP Echo Server v1 and 20% to v2.

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: destination
spec:
  host: tcp-echo-server
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 9000
      name: tcp
      protocol: TCP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: route
spec:
  hosts:
  - "*"
  gateways:
  - gateway
  tcp:
  - match:
    - port: 9000
    route:
    - destination:
        host: tcp-echo-server
        port:
          number: 9000
        subset: v1
      weight: 80
    - destination:
        host: tcp-echo-server
        port:
          number: 9000
        subset: v2
      weight: 20
```

### Deploying The Route Configuration

To apply the configuration, copy it to a file named `route-config.yaml` and install it via the following command.

```
kubectl apply -f route-config.yaml
destinationrule.networking.istio.io/destination created
gateway.networking.istio.io/gateway created
virtualservice.networking.istio.io/route created
```

### Verifying Istio’s TCP Routing Behavior

We forward requsts from a local system port to the `istio-ingressgateway` like so:

```
$ kubectl get pods -n istio-system | grep ingressgateway
istio-ingressgateway-7b9bff766d-l478h  1/1  Running  0  30m

$ kubectl port-forward istio-ingressgateway-7b9bff766d-l478h -n istio-system 9000 &
[1] 69266
Forwarding from 127.0.0.1:9000 -> 9000
Forwarding from [::1]:9000 -> 9000
```

We can now send a few requests to the weight balanced TCP Echo Server as shown below.

```
$ for i in {1..10}; do
for> docker run -it --rm busybox sh -c '(date; sleep 1) | nc docker.for.mac.localhost 9000'
for> done
one Sat Oct 20 04:38:05 UTC 2018
two Sat Oct 20 04:38:07 UTC 2018
two Sat Oct 20 04:38:09 UTC 2018
one Sat Oct 20 04:38:12 UTC 2018
one Sat Oct 20 04:38:14 UTC 2018
one Sat Oct 20 04:38:17 UTC 2018
one Sat Oct 20 04:38:19 UTC 2018
one Sat Oct 20 04:38:22 UTC 2018
one Sat Oct 20 04:38:24 UTC 2018
two Sat Oct 20 04:38:27 UTC 2018
```

As you see, about 80% of the requests have a prefix of “one” and the rest 20% have a prefix of “two”. This proves that the weighted TCP routes are indeed in effect.

The diagram below should give you a good idea as to how the Istio landscape for this demonstration looks like.

![Architecture](https://venilnoronha.io/assets/images/2018-10-19-raw-tcp-traffic-shaping-with-istio-1.1.0/architecture.svg)

## Cleanup

First, let’s stop the port forwarding via the following command.

```
$ killall kubectl
[1]  + 69266 terminated  kubectl port-forward istio-ingressgateway-7b9bff766d-l478h -n istio-system
```

We then delete the Minikube deployment, like so:

```
$ minikube stop && minikube delete
Stopping local Kubernetes cluster...
Machine stopped.
Deleting local Kubernetes cluster...
Machine deleted.
```

## Conclusion

As shown in this article, it’s quite easy to configure weighted TCP routes via the upcoming Istio 1.1.0 release. Also, this article should give you a good idea on constructing your own weighted TCP routes, and shaping your TCP traffic from the ground up!