![alt text][israel]

# Istio 不仅为微服务而生

通过使用 Istio Service Mesh 来保障 Kubernetes 平台服务。通常可以运行实例代码有助于用户更清晰的理解并将概念应用到实际的案例中。该项目围绕一个基本的 Node.js 应用程序演示了以 Istio Service Mesh 为 ETCD 的持久化数据服务的能力。

## Istio 背景
Istio 是一个 连接、管理以及保障微服务的开放平台。如需要了解更多 Istio 的信息，请访问 [Intro page]( https://istio.io/about/intro.html) 。

## 安装
假设已对 Kubernetes 有了初步了解。在这个项目中，有一组脚本，假定已预先安装了 Docker、Kubernetes CLI 以及JQ，用于操作 Kubernetes commands 返回的各种 JSON 对象。也有某种层度对 Node.js 知识的假设。

**各种工具的连接如下：.**  
**Docker 安装:** https://docs.docker.com/install/  
**Kubernetes 安装:** https://kubernetes.io/docs/tasks/tools/install-kubectl/   
**jq 下载地址:** https://stedolan.github.io/jq/download/     
**Node.js 下载地址:** https://nodejs.org/en/download/     

## Kubernetes Private
以下代码可以运行在任何符合 Kubernetes 的提供者上，并且已经在 Minikube 和私有 IBM Cloud上。根据选择的提供者的不同，指令会略有不同。

### Minikube
Minikube 可用与下载和安装的地址：[点我](https://kubernetes.io/docs/tasks/tools/install-minikube/)。Minikube 为学习 Kubernetes 提供了一个简单易用的开发环境。

### IBM 私有云
IBM 为开发者提供了其 Kubernetes 运行时的免费社区版，并包含了与企业版生产版本相同的大多数功能，高可用性例外。安装 IBM 私有云服务，请查看  [2.1.0安装向导](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_2.1.0/installing/install_containers_CE.html)

## Istio Index Conference 2018 Application
从代码入手, 可克隆如下仓库 ```git clone git@github.com:todkap/istio-index-conf2018.git```

### Kubernetes 安装
- **Minikube:** Prior to deploying to Minikube, Minikube first needs to be started.   In the root directory of this project, there is a script ```createMinikubeEnv.sh``` that tears down the previous Minikube environment and initializes a new environment with the appropriate Kubernetes context.

- **Minikube：** 请优先部署并第一个启动 Minikube 。在这个项目的根目录下，有一个脚本 ```createMinikubeEnv.sh``` ，

- **IBM Cloud Private:** IBM Cloud Private has a [configure client](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_2.1.0/manage_cluster/cfc_cli.html) step that will configure the Kubernetes CLI to point to a given IBM Cloud Private installation.  This context will be used each time the Kubectl CLI executes commands.

### Deploy
#### Kubernetes Installation 
This project contains a script that will deploy Istio and the application to Kubernetes called ```deploy.sh```.  The script provides verbose output as it progresses through the various steps waiting for the entire system to be in ```Running``` state prior to exiting.

#### Helm Installation
Starting with IBM Cloud Private version 2.1.0.3, the Istio Control Plane can be installed via a Helm chart as part of the initial install or via the Catalog post installation.  Included in this project is an additional script called ```icp-helm-deploy``` that leverages a combination of the IBM Cloud Private CLI, Helm CLI and Kubernetes CLI to install the Istio Index application.   In an effort to simplify the deployment process and promote some of the latest features of Istio, I have enabled  [automatic sidecar injection](https://istio.io/docs/setup/kubernetes/sidecar-injection.html#automatic-sidecar-injection) for this application. 

### Testing
This project contains two scripts for testing depending upon which Kubernetes provider that is used. The only difference in the two scripts is the setting of the ingress IP address for IBM Cloud Private.   To test choose either ```testICPEnv.sh``` or ```testMinikubeEnv.sh``` based upon your provider.

In addition to the scripts, there is a lightweight web interface for interacting with the REST APIs.   
![alt text][ui]

### Verification
To verify the success of the Istio integration, the script executes a set of tests.  

- The first test verifies a simple put test to the etcd service NodePort to validate connectivity to etcd.  
**Example output**
```
simple etcd test
*   Trying 192.168.64.20...
* TCP_NODELAY set
* Connected to 192.168.64.20 (192.168.64.20) port 32012 (#0)
> PUT /v2/keys/message HTTP/1.1
> Host: 192.168.64.20:32012
> User-Agent: curl/7.54.0
> Accept: */*
> Content-Length: 17
> Content-Type: application/x-www-form-urlencoded
> 
* upload completely sent off: 17 out of 17 bytes
< HTTP/1.1 201 Created
< content-type: application/json
< x-etcd-cluster-id: cdf818194e3a8c32
< x-etcd-index: 14
< x-raft-index: 15
< x-raft-term: 2
< date: Wed, 14 Feb 2018 19:45:24 GMT
< content-length: 102
< x-envoy-upstream-service-time: 1
< server: envoy
< x-envoy-decorator-operation: default-route
< 
{"action":"set","node":{"key":"/message","value":"Hello world","modifiedIndex":14,"createdIndex":14}}
* Connection #0 to host 192.168.64.20 left intact
```

- The second test verifies that the Node application can handle a simply ping request as well as proxy requests to etcd using the Node applications NodePort.  
**Example output**
```
-------------------------------
simple ping test
*   Trying 192.168.64.20...
* TCP_NODELAY set
* Connected to 192.168.64.20 (192.168.64.20) port 32380 (#0)
> GET / HTTP/1.1
> Host: 192.168.64.20:32380
> User-Agent: curl/7.54.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< x-powered-by: Express
< content-type: text/html; charset=utf-8
< content-length: 46
< etag: W/"2e-FL84XHNKKzHT+F1kbgSNIW2RslI"
< date: Wed, 14 Feb 2018 19:45:24 GMT
< x-envoy-upstream-service-time: 1
< server: envoy
< x-envoy-decorator-operation: default-route
< 
* Connection #0 to host 192.168.64.20 left intact
Simple test for liveliness of the application!
-------------------------------
test etcd service API call from node app
*   Trying 192.168.64.20...
* TCP_NODELAY set
* Connected to 192.168.64.20 (192.168.64.20) port 32380 (#0)
> PUT /storage HTTP/1.1
> Host: 192.168.64.20:32380
> User-Agent: curl/7.54.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 60
> 
* upload completely sent off: 60 out of 60 bytes
< HTTP/1.1 201 Created
< x-powered-by: Express
< date: Wed, 14 Feb 2018 19:45:24 GMT
< x-envoy-upstream-service-time: 12
< server: envoy
< x-envoy-decorator-operation: default-route
< transfer-encoding: chunked
< 
* Connection #0 to host 192.168.64.20 left intact
nodeAppTesting created(etcd-service) ->{"key":"istioTest","value":"Testing Istio using NodePort"}:{"action":"set","node":{"key":"/istioTest","value":"Testing Istio using NodePort","modifiedIndex":15,"createdIndex":15},"prevNode":{"key":"/istioTest","value":"Testing Istio using Ingress","modifiedIndex":13,"createdIndex":13}}
*   Trying 192.168.64.20...
* TCP_NODELAY set
* Connected to 192.168.64.20 (192.168.64.20) port 32380 (#0)
> GET /storage/istioTest HTTP/1.1
> Host: 192.168.64.20:32380
> User-Agent: curl/7.54.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< x-powered-by: Express
< date: Wed, 14 Feb 2018 19:45:24 GMT
< x-envoy-upstream-service-time: 14
< server: envoy
< x-envoy-decorator-operation: default-route
< transfer-encoding: chunked
< 
* Connection #0 to host 192.168.64.20 left intact
nodeAppTesting(etcd-service) ->{"action":"get","node":{"key":"/istioTest","value":"Testing Istio using NodePort","modifiedIndex":15,"createdIndex":15}}
-------------------------------
```

- The next level of tests start to test Istio where traffic is routed through the Istio Ingress then to the Node application.  
**Example output**
```
simple hello test
*   Trying 192.168.64.20...
* TCP_NODELAY set
* Connected to 192.168.64.20 (192.168.64.20) port 32612 (#0)
> GET / HTTP/1.1
> Host: 192.168.64.20:32612
> User-Agent: curl/7.54.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< x-powered-by: Express
< content-type: text/html; charset=utf-8
< content-length: 46
< etag: W/"2e-FL84XHNKKzHT+F1kbgSNIW2RslI"
< date: Wed, 14 Feb 2018 19:45:24 GMT
< x-envoy-upstream-service-time: 6
< server: envoy
< 
* Connection #0 to host 192.168.64.20 left intact
Simple test for liveliness of the application!
-------------------------------
test etcd service API call from node app
*   Trying 192.168.64.20...
* TCP_NODELAY set
* Connected to 192.168.64.20 (192.168.64.20) port 32612 (#0)
> PUT /storage HTTP/1.1
> Host: 192.168.64.20:32612
> User-Agent: curl/7.54.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 59
> 
* upload completely sent off: 59 out of 59 bytes
< HTTP/1.1 201 Created
< x-powered-by: Express
< date: Wed, 14 Feb 2018 19:45:24 GMT
< x-envoy-upstream-service-time: 15
< server: envoy
< transfer-encoding: chunked
< 
* Connection #0 to host 192.168.64.20 left intact
nodeAppTesting created(etcd-service) ->{"key":"istioTest","value":"Testing Istio using Ingress"}:{"action":"set","node":{"key":"/istioTest","value":"Testing Istio using Ingress","modifiedIndex":16,"createdIndex":16},"prevNode":{"key":"/istioTest","value":"Testing Istio using NodePort","modifiedIndex":15,"createdIndex":15}}
*   Trying 192.168.64.20...
* TCP_NODELAY set
* Connected to 192.168.64.20 (192.168.64.20) port 32612 (#0)
> GET /storage/istioTest HTTP/1.1
> Host: 192.168.64.20:32612
> User-Agent: curl/7.54.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< x-powered-by: Express
< date: Wed, 14 Feb 2018 19:45:24 GMT
< x-envoy-upstream-service-time: 13
< server: envoy
< transfer-encoding: chunked
< 
* Connection #0 to host 192.168.64.20 left intact
nodeAppTesting(etcd-service) ->{"action":"get","node":{"key":"/istioTest","value":"Testing Istio using Ingress","modifiedIndex":16,"createdIndex":16}}
-------------------------------
```

- The final set of tests grep the istio-proxy logs searching for access logs for the client and server proxies to validate the traffic is routed through Istio.  
**Example output**
```
client logs from istio-proxy
[2018-02-14T16:28:24.640Z] "PUT /v2/keys/istioTest HTTP/1.1" 201 - 40 119 6 5 "-" "-" "45dd6431-49cf-9bcf-b611-1d319c56eb2e" "etcd-service:2379" "172.17.0.9:2379"
[2018-02-14T16:28:24.672Z] "GET /v2/keys/istioTest HTTP/1.1" 200 - 0 119 3 3 "-" "-" "8aa0f7d8-caac-9065-bb4c-d11c7af7d93f" "etcd-service:2379" "172.17.0.9:2379"
server logs from istio-proxy
[2018-02-14T16:28:24.640Z] "PUT /v2/keys/istioTest HTTP/1.1" 201 - 40 119 4 1 "-" "-" "45dd6431-49cf-9bcf-b611-1d319c56eb2e" "etcd-service:2379" "127.0.0.1:2379"
[2018-02-14T16:28:24.673Z] "GET /v2/keys/istioTest HTTP/1.1" 200 - 0 119 3 0 "-" "-" "8aa0f7d8-caac-9065-bb4c-d11c7af7d93f" "etcd-service:2379" "127.0.0.1:2379"
```

### Istio Metrics 
Istio provides native enablement for capturimg metrics for network activity.  To drive some additional metrics, a simple test script can be run to populate the charts for both Grafana and Promotheus similar to the code below.   

**Test Script**  
```
## Simple load test using loadtest (https://www.npmjs.com/package/loadtest)
if [ -x "$(command -v loadtest)" ]; then
	loadtest -n 400 -c 10 --rps 20 http://$ingressIP:$ingressPort/storage/istioTest
	loadtest -n 400 -c 10 --rps 10 http://$ingressIP:$ingressPort/storage/istioTest
	loadtest -n 400 -c 10 --rps 40 http://$ingressIP:$ingressPort/storage/istioTest
fi
```

#### Grafana  
In your Kubernetes environment, execute the following command:
```
kubectl -n istio-system port-forward $(kubectl -n istio-system get \
   pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
```
Visit http://localhost:3000/dashboard/db/istio-dashboard in your web browser.  The Istio Dashboard will look similar to:

![alt text][grafana]

#### Prometheus
In your Kubernetes environment, execute the following command:
```
kubectl -n istio-system port-forward $(kubectl -n istio-system get \
    pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 &   
```
Visit http://localhost:9090/graph in your web browser. The Istio Dashboard will look similar to:

![alt text][prometheus]

#### Weave Scope 
During the deploy script, Weave Scope was also deployed to the environment.   In the console, the port for Weave Scope is logged but is also available using the command.
```
kubectl get service weave-scope-app --namespace=weave -o 'jsonpath={.spec.ports[0].nodePort}'; echo ''  
```
Weave Scope provides a Service Graph which will display the request flow for the tests executed in the test process. The Weave Scope Dashboard will look similar to:

![alt text][weavescope]

#### Kiali  
Kiali is a relatively new project focused on Service Mesh Observability and supports Istio 0.7.1 or later.  Inside of this project is a separate script ```setupKiali.s``` that will build and install Kiali as well as apply the appropriate ClusterRole required to run on IBM Cloud Private.  To view the console in your environemnt, you will need the NodePort for the servivce.  To retrieve the port for Kiali  use the following command.
```
kubectl get service kiali --namespace=istio-system -o 'jsonpath={.spec.ports[0].nodePort}'; echo ''  
```
Kiali provides a Service Graph similar to Weave Scope capable of showing historical view of the request flow as well as other interesting views of your K8 environment such as Services and Tracing. To see this capability in action, the service graph should be viewed after the load test scripts are executed. The Kiali Dashboard will look similar to:

![alt text][kiali]



### Slides
**Istio is not just for microservices on Slideshare:** https://www.slideshare.net/ToddKaplinger/istio-is-not-just-for-microservices


### Notes
- This project is based upon a Medium Article [Istio is not just for microservices](https://medium.com/ibm-cloud/istio-is-not-just-for-microservices-4ed199322bf4) written in 2017 and updated to support the latest version of Istio and Kubernetes.   Since most of the content was embedded within the original Medium article, this project was created to encourage developers to clone this repository and modify it to learn more about Kubernetes, Istio and etcd.
- The Node.js application source code is included in the nodejs subdirectory of the project and also includes the Dockerfile and build script for deploying to a Docker registry.   Some modifications would be required to publish the image to your own Docker registry and to have the deployment yaml reference the new image but should be relatively easy to figure out if necessary.

[grafana]: https://github.com/todkap/istio-index-conf2018/blob/master/images/loadtest_grafana.png "Load Test Grafana"
[prometheus]: https://github.com/todkap/istio-index-conf2018/blob/master/images/loadtest_prometheus.png "Load Test Prometheus"
[weavescope]: https://github.com/todkap/istio-index-conf2018/blob/master/images/weavescope.png "Weave Scope"
[kiali]: https://github.com/todkap/istio-index-conf2018/blob/master/images/loadtest_kiali.png "Kiali Console"


[israel]: https://github.com/todkap/istio-index-conf2018/blob/master/images/1397375_910870955327_147651637_o.jpg "Israel"
[ui]: https://github.com/todkap/istio-index-conf2018/blob/master/images/ui.png "Simple Form"

