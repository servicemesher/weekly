---
original: https://medium.com/pismolabs/istio-envoy-grpc-metrics-winning-with-service-mesh-in-practice-d67a08acd8f7
translator: shaobai
reviewer: 
title: ""
description: ""
categories: "译文"
tags: ["Envoy","服务网格","Service Mesh"]
date: 2018-11-02
---

Amazing times we’re living when one can set up an entire environment with applications intelligently integrating, gathering metrics without having to write a single line of code.

In this post I will demonstrate how to setup an Istio mtls environment with helm, the necessary configuration yamls and other perks you gain out of the box by installing Istio. In addition, at the end I will also show some examples of routing configurations.

I’m assuming that Kubernetes and helm are already installed and that you have some previous knowledge with them. This tutorial will use AWS as our environment. More on the official docs:
[https://istio.io/docs/setup/kubernetes/helm-install/](https://istio.io/docs/setup/kubernetes/helm-install/)

Below is the official image that shows the topology of Istio

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fwymh3xq3ij30z00o6abc.jpg)

With our setup all containers that are deployed in the istio-injected namespace will be created with an istio-proxy injected on it. The application will then talk to the istio-proxy (envoy) which will handle all the connections, mtls and load balancing with other services.

## **Installation and Configuration**

For starters, download and unzip istio-1.0.0

```
wget https://github.com/istio/istio/archive/1.0.0.zip\ && unzip 1.0.0.zip
```

Now we will modify the values.yaml in the istio folder to adjust some settings necessary to our environment

```
istio-1.0.0/install/kubernetes/helm/istio/values.yaml
```

The following modifications will enable mtls, ingress (for monitoring services) and ssl port (443) from istio ingressgateway to redirect to port 80 and enable all the addons to also be installed (grafana, servicegraph, jaeger and kiali):

Click [here](https://gist.github.com/Stocco/63077eab8cfd116aeb591fdb9958413c) to get the modified values.yaml

With the modifications made, let’s install everything and see Istio running!

First, If using a Helm version prior to 2.10.0, install Istio’s [Custom Resource Definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions) via kubectl apply ; if your helm doesn't fit the version skip this step.

```
kubectl apply -f istio-1.0.0/install/kubernetes/helm/istio/templates/crds.yaml
kubectl apply -f istio-1.0.0/install/kubernetes/helm/istio/charts/certmanager/templates/crds.yaml
```

Now install Istio (**remember to change the values.yaml file to our modified one!**)

```
helm install istio-1.0.0/install/kubernetes/helm/istio --name istio --namespace istio-system
```

Check your istio-system namespace and see if the pods are showing up there! Now create an namespace to your applications:

```
kubectl create namespace istio-apps
```

Now label it so that Istio knows where to inject the istio-proxies

```
kubectl label namespace istio-apps istio-injection=enabled
```

Now that we have our environment running lets recap how the applications will talk to each other. All the services inside the mesh will communicate with each other via it’s services with envoy handling the mtls and the load balancing. What about services that are not in the mesh? You may wonder how they can talk to our applications and our applications contact other external services. That’s where our configuration comes in. Here is an image to illustrate how external services will contact our applications within the mesh:

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fwymltt2dxj31ff0quq4b.jpg)

All external traffic will enter via the Istio-ingressgateway that will try to find an virtual service that matches the host and path corresponding to an service within the mesh. If no virtual services are present, no external service can reach the applications inside the mesh.

**Tip: It is in the virtualservice that we will be making our traffic management handling!**

For services inside the mesh that want to reach outside services like the application stocks in the image, you must also map the traffics with serviceentries.

For this tutorial we will enable all hosts to be reach from within the mesh with the following yaml (you can also specify ips):

```
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: allhosts-serviceentry
  namespace: istio-apps
spec:
  hosts:
  - "*"
  ports:
  - number: 80
    name: http
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  - number: 3306
    name: tcp
    protocol: TCP
```

Now we must make sure that our istio-ingressgateway is mapped in the istio-apps namespace. First, checkout your istio-ingressgateway service and create an cname domain pointing to the load balancer that was created by that service.

We will now map the cname you created in the istio-ingressgateway with this yaml, for this tutorial we will use *.yourdomain. In production you should map your hosts one by one (this change can take some minutes to apply):

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-gateway
  namespace: istio-apps
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*.yourdomain"
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: PASSTHROUGH
    hosts:
    - "*.yourdomain"
```
istio-gateway

Now we are all set to test our environment. I’ve created an application that servers http(8080)/grpc(8333) and dials to a second application to test our environment. I also created the deployment and service files to bootstart our test. Copy the the following yaml and apply it to your environment:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: application
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: application
        app: application
        version: 1.0.0
    spec:
      serviceAccountName: default
      containers:
      - name: application
        image: pismotest/istio-application:v1
        ports:
        - containerPort: 8080
          name: http-port
        - containerPort: 8333
          name: grpc-port
---
apiVersion: v1
kind: Service
metadata:
  name: application
spec:
  ports:
    - name: http
      port: 8080
      targetPort: 8080
    - name: grpc-port
      port: 8333
      targetPort: 8333
  selector:
    name: application
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: applicationtwo
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: applicationtwo
        app: applicationtwo
        version: 1.0.0
    spec:
      serviceAccountName: default
      containers:
      - name: applicationtwo
        image: pismotest/istio-application:v1
        ports:
        - containerPort: 8080
          name: http-port
        - containerPort: 8333
          name: grpc-port
---
apiVersion: v1
kind: Service
metadata:
  name: applicationtwo
spec:
  ports:
    - name: http
      port: 8080
      targetPort: 8080
    - name: grpc-port
      port: 8333
      targetPort: 8333
  selector:
    name: applicationtwo
```

```
use kubectl apply -f applications.yaml -n istio-apps
```

Now we will create our virtual service to map one of our applications so that the istio-ingressgateway can route the traffic to our application (**change application.yourdomain to your cname**).

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: application-ingress
  namespace: istio-apps
spec:
  hosts:
  - "application.yourdomain"
  gateways:
  -  istio-gateway
  http:
  - match:
    - uri:
        prefix: /health
    route:
    - destination:
        port:
          number: 8080
        host: application
    retries:
      attempts: 25
      perTryTimeout: 1s
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        port:
          number: 8333
        host: application
    retries:
      attempts: 25
      perTryTimeout: 1s
```

Virtual services will map the host to any service you want to match, in our case, the first rule we will try to match to the service http health endpoint, if that match fails we will try to redirect to our grpc service port. The retries part will aid us whenever there is some interference on the network or when there are unhealthy pods. You can add as many matches as you want, you can also match any other form of request to the host with this rule:

```
 - match:
     - uri:
         regex: ".+"
```

You should be able to reach your /health endpoint. Try it:

```
curl --request GET \
  --url http://application.yourdomain/health
```

You should receive 200, and your application should log the request received.**If you receive 404, probably your virtualservice is not mapping your request uri to your service!**

Try now to propagate:

```
curl --request GET \
  --url http://application.yourdomain/health \
  --header 'propagate: yes'
```

Your application should now send requests to application two. Check application two logs.

Piece of cake! Istio also maps the requests made with grpc with the same rule. Clone the application repository that I used to make these applications, change the domain in the main.go dial function so you can try out!

[https://github.com/Stocco/istioapplications](https://github.com/Stocco/istioapplications)

## **Visualizing mesh metrics**

There are many ways to visualize what is going inside the mesh, I will list some of them in this section.

Note that Istio will gather all the necessary metrics to plot the graphics and stuff, but for some metrics applications like **Kiali and Jaeger** you must ensure that your application is propagating the Istio injected headers in all requests made by it, so these applications can tie together the history of the request.

Check the example in the application handle health function:

[https://github.com/Stocco/istioapplications/blob/a3c3275a63a0667f870d054ea5940284b8a100af/main.go#L72](https://github.com/Stocco/istioapplications/blob/a3c3275a63a0667f870d054ea5940284b8a100af/main.go#L72)

## **Kiali**

**Kiali** will help us see what is going on in real time. Expose the kiali port locally so you can see the application:

```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') 20001:20001
```

Now click this link and login with admin(username)/admin(password):

```
http://localhost:20001/console/service-graph/istio-apps?layout=cose-bilkent&duration=10800&edges=responseTime95thPercentile&graphType=versionedApp
```

You should see this page:

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fwymwetflpj31es0opmy8.jpg)

Nice, right? There are many more tools in Kiali that you should be checking out!

## **Jaeger**

Jaeger is a powerful tool to monitor what is going on with the requests and how long each part of the request is consuming. However, keep in mind that in order to use it’s full potential, you will need to adapt your code a little by propagating the Istio injected headers. If you want to know more about the requests you are going to need some tools (such as opentracing) to get metrics about how long the inner functions of your app is taking.

Expose jaeger port and access [http://localhost:16686](http://localhost:16686)

```
kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686
```

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fwymxngfikj31eb0o940d.jpg)

You have some filters on the left, choose whatever you want, you will get the latest traces by clicking on find traces. Choose one of the traces and you will see the exact time consumed by each request of your application!

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fwymxyb2c7j31ez0i9taf.jpg)

It could be even better if the application has opentracing installed and properly used in each function call!

## **More metric tools**

There are more 3 applications that are already installed with our values.yaml that use all the metrics gathered by Istio. Try checking them, they are grafana, prometheus and servicegraph.

Expose them with the following commands:

```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 

kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &

kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=servicegraph -o jsonpath='{.items[0].metadata.name}') 8088:8088 &

Tips:
  for this tutorial grafana login/password is set admin/admin (change it on values.yaml)
  when exposing servicegraph use this url to see the magic:
  http://localhost:8088/force/forcegraph.html?time_horizon=3000s&filter_empty=true
```

## **Intelligent routing**

What if you could manage your applications versioning? Istio can provide this with some minor changes to your virtualservice.yaml. Lets adapt our virtual service (**change application.yourdomain to your cname**):

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: application-ingress
  namespace: istio-apps
spec:
  hosts:
  - "application.yourdomain"
  gateways:
  -  istio-gateway
  http:
  - match:
    - headers:
        myself:
          exact: "yourself"
    route:
    - destination:
        port:
          number: 8080
        host: applicationtwo
  - match:
    - uri:
        prefix: /health
    route:
    - destination:
        port:
          number: 8080
        host: application
    retries:
      attempts: 25
      perTryTimeout: 1s
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        port:
          number: 8333
        host: application
    retries:
      attempts: 25
      perTryTimeout: 1s  
```

The rules follow a precedence order. Now try using the same curl from the previous section with the header we have put on the first rule:

```
curl --request GET \
  --url http://application.pismolabs.io/health \
  --header 'myself: yourself'
```

You should see the log from application two instead of the first application! And that is because the first rule that matched was applied.

## **Grpc ssl destination rule**

If you are working or worked with grpc you probably know that applying ssl to your grpc calls is a problem. Mostly because you have to put the certificate of the server in your code to make the ssl tunnel magic happen. What if you could achieve that only by setting some configurations in the infrastructure?

Try creating this service-entry and this destination rule for your domain and see your non-ssl application calls your external grpc service encrypted!

```
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-grpc-service-entry
spec:
  hosts:
  - application.yourdomain
  ports:
  - number: 8333
    name: grpc-ssl
    protocol:  GRPC
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: originate-tls-grpc
spec:
  host: application.yourdomain
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    portLevelSettings:
    - port:
        number: 8333
      tls:
        mode: SIMPLE
```

There are many serviceentries + destination rules options that you can combine to create a richer environment to your applications. Checkout the docs for more.

```
https://istio.io/docs/reference/config/istio.networking.v1alpha3/#DestinationRule
```

## **We’re just scratching the surface**

Congratulations! You got your first Istio environment applications working. Now, try swapping my deployment.yaml with your own application and see it working in Istio.

There are more perks you can get with Istio, like special destination rules, custom policies and custom metrics but they are subjects for the next posts.

Feel free to add any comments and reach out to me if you have any doubts, suggestions or feedback from this tutorial!

See you in the community!