---
original: https://blog.jdriven.com/2018/11/transcoding-grpc-to-http-json-using-envoy/
translator: malphi
reviewer: rootsongjc
title: "使用Envoy转码gRPC到HTTP/JSON"
description: "本文用实例讲解了如何利用Envoy将gRPC转码为HTTP/JSON"
categories: "译文"
tags: ["Envoy"]
date: 2018-11-15
---

# 使用Envoy将gRPC转码为HTTP/JSON

在gRPC中构建服务时，要在.proto文件中定义消息和服务。gRPC支持多种语言自动生成客户端、服务器和DTO实现。这篇文章的最后，您将了解到使用Envoy作为转码代理，使gRPC API也可以通过HTTP JSON访问。您可以通过github代码库中的Java代码来测试它。有关gRPC的快速介绍请阅读[blog.jdriven.com/2018/10/grpc-as-an-alternative-to-rest/](https://blog.jdriven.com/2018/10/grpc-as-an-alternative-to-rest/).

## **为什么转码gRPC服务？**

一旦有了一个可用的gRPC服务，您可以通过向服务添加一些额外的注解将gRPC服务作为HTTP JSON API发布。然后需要一个代理来转换HTTP JSON调用并将其传递给gRPC服务。我们称这个过程为转码。然后你的服务可以通过gRPC和HTTP/JSON访问。大多数时候我更倾向使用gRPC，因为使用遵循“契约”生成的类型安全的代码更方便、更安全，但有时转码也很有用：

1. 您的web应用程序可以通过HTTP/JSON调用与gRPC服务通信。[github.com/grpc/grpc-web](https://github.com/grpc/grpc-web)是一个可以在浏览器中使用的JavaScript的gRPC实现。这个项目很有前途，但还不成熟。
2. 因为gRPC在网络上使用二进制格式，所以很难看到实际发送和接收的内容。将其作为HTTP/JSON API公开，可以通过使用cURL或postman等工具更容易地检查服务。
3. 如果您使用的语言gRPC不支持，您可以通过HTTP/JSON访问它。
4. 它为在项目中更平稳地采用gRPC铺平了道路，允许其他团队逐步过渡。

## **创建一个gRPC服务：ReservationService**

让我们创建一个简单的gRPC服务作为示例。在gRPC中，定义包含远程过程调用(rpc)的类型和服务。您可以随意设计自己的服务，但是谷歌建议使用面向资源的设计(源代码：[cloud.google.com/apis/design/resources](https://cloud.google.com/apis/design/resources))，因为用户无需知道每个方法是做什么的就可以容易地理解API。如果您创建了许多松散格式的rpc，您的用户必须理解每种方法的作用，从而使您的API更难学习。面向资源的设计还可以更好地转换为HTTP/JSON API。

在本例中，我们将创建一个会议预订服务。该服务称为ReservationService，由创建、获取、列出和删除预订4个操作组成。这是服务定义:

```protobuf
//reservation_service.proto
 
syntax = "proto3";
 
package reservations.v1;
option java_multiple_files = true;
option java_outer_classname = "ReservationServiceProto";
option java_package = "nl.toefel.reservations.v1";
 
import "google/protobuf/empty.proto";
 
service ReservationService {
 
    rpc CreateReservation(CreateReservationRequest) returns (Reservation) {  }
    rpc GetReservation(GetReservationRequest) returns (Reservation) {  }
    rpc ListReservations(ListReservationsRequest) returns (stream Reservation) {  }
    rpc DeleteReservation(DeleteReservationRequest) returns (google.protobuf.Empty) {  }
 
}
 
message Reservation {
    string id = 1;
    string title = 2;
    string venue = 3;
    string room = 4;
    string timestamp = 5;
    repeated Person attendees = 6;
}
 
message Person {
    string ssn = 1;
    string firstName = 2;
    string lastName = 3;
}
 
message CreateReservationRequest {
    Reservation reservation = 2;
}
 
message CreateReservationResponse {
    Reservation reservation = 1;
}
 
message GetReservationRequest {
    string id = 1;
}
 
message ListReservationsRequest {
    string venue = 1;
    string timestamp = 2;
    string room = 3;
 
    Attendees attendees = 4;
 
    message Attendees {
        repeated string lastName = 1;
    }
}
 
message DeleteReservationRequest {
    string id = 1;
}
```

It is common practice to wrap the input for the operations inside a request object. This makes adding extra fields or options to your operation in the future easier. The ListReservations operation returns a stream of Reservations. In Java that means you will get an iterator of Reservation objects. The client can start processing the responses before the server is even finished sending them, pretty awesome :D.

通常的做法是将操作的输入包装在请求对象中。这使得在以后的操作中添加额外的字段或选项更加容易。ListReservations操作返回一个Reservations流。在Java中，这意味着您将得到Reservations对象的迭代器。客户端甚至可以在服务器发送完响应之前就开始处理它们，非常棒:D。

If you would like to see how this gRPC service can be used in Java, see如果你想知道这个gRPC服务在Java中是如何使用的，请查看 [ServerMain.java](https://github.com/toefel18/transcoding-grpc-to-http-json/blob/master/src/main/java/nl/toefel/server/ServerMain.java) 和 [ClientMain.java](https://github.com/toefel18/transcoding-grpc-to-http-json/blob/master/src/main/java/nl/toefel/client/ClientMain.java)实现。

## **使用HTTP选项对服务进行注解来转码**

在每个rpc操作的花括号中可以添加选项。Google定义了一个java option，允许您指定如何将操作转换到HTTP端点。在*reservation_service.proto*中引入 ‘**google/api/annotations.proto’**即可使用该选项。在默认情况下这个import是不可用的，但是您可以通过向*build.gradle*添加以下编译依赖来实现它：

```gradle
compile "com.google.api.grpc:proto-google-common-protos:1.13.0-pre2"
```

这个依赖将由protobuf解压并将几个.proto文件放入构建目录中。现在可以把**\*google/api/annotations.proto***引入你的.proto文件中并开始指定如何转换API。

## **转码GetReservation操作为GET方法**

让我们从GetReservation操作开始，我已经添加了GetReservationRequest到代码示例中：

```protobuf
  message GetReservationRequest {
       string id = 1;
   }
 
   rpc GetReservation(GetReservationRequest) returns (Reservation) {
       option (google.api.http) = {            
           get: "/v1/reservations/{id}"         
       };
   }
```

在选项定义中有一个名为“get”的字段，设置为“/v1/reservation /{id}”。字段名对应于HTTP客户端应该使用的HTTP请求方法。get的值对应于请求URL。在URL中，我们看到一个名为id的路径变量，这个路径变量会自动映射到输入操作中同名的字段。在本例中，它将是GetReservationRequest.id。

发送 **GET /v1/reservations/1234** 到代理将转码到下面的伪代码：

```java
var request = GetReservationRequest.builder().setId(“1234”).build()
var reservation = reservationServiceClient.GetReservation(request)
return toJson(reservation)
```

HTTP response body将返回预订的所有非空字段的JSON形式。

**记住：转码不是由gRPC服务完成的。单独运行这个示例不会将其发布为HTTP JSON API。前端的代理负责转码。我们稍后将对此进行配置。**

## 转码CreateReservation操作为POST方法

现在来考虑CreateReservation操作。

```protobuf
message CreateReservationRequest {
   Reservation reservation = 2;
}
 
rpc CreateReservation(CreateReservationRequest) returns (Reservation) {
   option(google.api.http) = {
      post: "/v1/reservations"
      body: "reservation"
   };
}
```

这个操作被转为POST请求*/v1/reservation*。选项中的body字段告诉转码器将请求体转成CreateReservationRequest中的字段。这意味着我们可以使用以下curl调用：

```shell
curl -X POST \
    http://localhost:51051/v1/reservations \
    -H 'Content-Type: application/json' \
    -d '{
    "title": "Lunchmeeting",
    "venue": "JDriven Coltbaan 3",
    "room": "atrium",
    "timestamp": "2018-10-10T11:12:13",
    "attendees": [
       {
           "ssn": "1234567890",
           "firstName": "Jimmy",
           "lastName": "Jones"
       },
       {
           "ssn": "9999999999",
           "firstName": "Dennis",
           "lastName": "Richie"
       }
    ]
}'
```

响应包含同样的对象，只不过多了一个生成的id字段。

## **转码带查询参数过滤的ListReservations**

查询集合资源的一种常见方法是提供查询参数作为过滤器。ListReservations的gRPC服务就有此功能。ListReservations接收到一个包含可选字段的ListReservationRequest，用于过滤预订集合。

```protobuf
message ListReservationsRequest {
    string venue = 1;
    string timestamp = 2;
    string room = 3;
 
    Attendees attendees = 4;
 
    message Attendees {
        repeated string lastName = 1;
    }
}
 
rpc ListReservations(ListReservationsRequest) returns (stream Reservation) {
   option (google.api.http) = {
       get: "/v1/reservations"
   };
}
```

在这里，转码器将自动创建ListReservationsRequest，并将查询参数映射到ListReservationRequest的内部字段。没有指定的所有字段都包含默认值，对于字符串是""。例如:

```shell
curl http://localhost:51051/v1/reservations?room=atrium
```

字段room设置为atrium并映射到ListReservationRequest里，其余字段设置为默认值。还可以提供以下子消息字段：

```shell
curl "http://localhost:51051/v1/reservations?attendees.lastName=Richie"
```

attendees.lastName是一个repeated的字段，可以被设置多次：

```shell
curl  "http://localhost:51051/v1/reservations?attendees.lastName=Richie&attendees.lastName=Kruger"
```

gRPC服务将会看到ListReservationRequest.attendees.lastName作为一个列表有两个元素：Richie和Kruger. Supernice。

## **运行转码器**

是时候让这些运行起来了。Google cloud支持转码，即使运行在Kubernetes (incl GKE) 或计算引擎中。更多信息请参看[cloud.google.com/endpoints/docs/grpc/tutorials](https://cloud.google.com/endpoints/docs/grpc/tutorials).

If you are not running inside the Google cloud or when you’re running locally, then you can use Envoy. Envoy is a very flexible proxy initially created by Lyft. It’s a major component in [istio.io](https://istio.io/) as well. We will use Envoy for this example.

如果您不在Google cloud中运行，或者在本地运行，那么您可以使用Envoy。它是一个由Lyft创建的非常灵活的代理。它也是[istio.io](https://istio.io/)中的主要组件。在这个例子中，我们将使用Envoy。

为了转码我们需要：

1. 一个gRPC服务的项目，在.proto文件中包含转码选项。
2. 从.proto文件中生成的.pd文件包含gRPC服务描述。
3. 使用该定义，配置Envoy作为gRPC服务的HTTP请求代理。
4. 使用docker运行Envoy。

### **STEP 1**

我已经创建了如上描述的项目并发布在github上。你可以从这里clone： [github.com/toefel18/transcoding-grpc-to-http-json](https://github.com/toefel18/transcoding-grpc-to-http-json)。然后构建：

```shell
# Script will download gradle if it’s not installed, no need to install it :)
./gradlew.sh clean build    # windows: ./gradlew.bat clean build
```

**提示：我创建了脚本自动执行步骤2到4，在项目[github.com/toefel18/transcoding-grpc-to-http-json](github.com/toefel18/transcoding-grpc-to-http-json)的根目录下。这将节省你的开发时间。步骤2到4详细的解释了它是如何工作的。**

```shell
./start-envoy.sh
```

### **STEP 2**

Then we need to create .pb file. To do this, you need to download the precompiled protoc executable here [github.com/protocolbuffers/protobuf/releases/latest](https://github.com/protocolbuffers/protobuf/releases/latest)(choose the correct version for your platform, i.e. *protoc-3.6.1-osx-x86_64.zip* for mac) and extract it somewhere in your path, that’s it, simple.

Then running the following command inside the [transcoding-grpc-to-http-json](https://github.com/toefel18/transcoding-grpc-to-http-json)directory will result in a *reservation_service_definition.pb* file that envoy understands. (Don’t forget to build the project first to actually retrieve the required *.proto* files imported by *reservation_service.proto*)



| 1234 | protoc -I. -Ibuild/extracted-include-protos/main --include_imports \               --include_source_info \               --descriptor_set_out=reservation_service_definition.pb \               src/main/proto/reservation_service.proto |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

This command might look complex but in reality it’s quite simple. The -I stands for include, these are directories where protoc will look for .proto files. *–descriptor_set_out* signifies the output file containing the definition and the last argument is the proto file we want to process.

### **STEP 3**

We are almost there, the last thing we need before running Envoy is to create its configuration file. Envoy’s configuration is described in yaml. There are many things you can do with Envoy, however, let’s now just focus on the minimum required to transcode our service. I took a basic config example from [their website](https://www.envoyproxy.io/docs/envoy/latest/configuration/http_filters/grpc_json_transcoder_filter#config-http-filters-grpc-json-transcoder)modified it a bit and marked the interesting parts with # markers.



| 123456789101112131415161718192021222324252627282930313233343536373839404142434445464748 | admin:  access_log_path: /tmp/admin_access.log  address:    socket_address: { address: 0.0.0.0, port_value: 9901 }         #1 static_resources:  listeners:  - name: main-listener    address:      socket_address: { address: 0.0.0.0, port_value: 51051 }      #2    filter_chains:    - filters:      - name: envoy.http_connection_manager        config:          stat_prefix: grpc_json          codec_type: AUTO          route_config:            name: local_route            virtual_hosts:            - name: local_service              domains: ["*"]              routes:              - match: { prefix: "/", grpc: {} }                  #3 see next line!                route: { cluster: grpc-backend-services, timeout: { seconds: 60 } }             http_filters:          - name: envoy.grpc_json_transcoder            config:              proto_descriptor: "/data/reservation_service_definition.pb" #4              services: ["reservations.v1.ReservationService"]            #5              print_options:                add_whitespace: true                always_print_primitive_fields: true                always_print_enums_as_ints: false                preserve_proto_field_names: false                        #6          - name: envoy.router   clusters:  - name: grpc-backend-services                  #7    connect_timeout: 1.25s    type: logical_dns    lb_policy: round_robin    dns_lookup_family: V4_ONLY    http2_protocol_options: {}    hosts:    - socket_address:        address: 127.0.0.1                       #8        port_value: 53000 |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
|                                                              |                                                              |

I’ve added some markers in the config file to emphasize the sections that are interesting to us:

- \#1 The address of the admin interface. You can also get prometheus metrics here to see how the service performs!!
- \#2 The address at which the HTTP API will be available
- \#3 The name of the backend services to route requests to. Step #7 defines this name.
- \#4 The path to the .pb descriptor file we generated before
- \#5 The services to transcode
- \#6 Protobuf field names usually contain underscores. Setting this field to false will translate field names to camelcase.
- \#7 A cluster defines upstream services (services that envoy can proxy to in step #3)
- \#8 The address and port at which the backend services are reachable. I’ve used (127.0.0.1/localhost).

### **STEP 4**

We are now ready to run envoy. The easiest way to run envoy is by running the docker image. This requires that docker is installed. If you haven’t, please [install docker](https://docs.docker.com/install/) first.

There are two resources that Envoy needs, the config file, and .pb descriptor file. We can map these files inside the container so that envoy finds them when it starts. Run this command within the github repo root directory:



| 1234 | sudo docker run -it --rm --name envoy --network="host" \  -v "$(pwd)/reservation_service_definition.pb:/data/reservation_service_definition.pb:ro" \  -v "$(pwd)/envoy-config.yml:/etc/envoy/envoy.yaml:ro" \  envoyproxy/envoy |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

If envoy started successfully you will see a log line at the end :



| 1    | [2018-11-10 14:55:02.058][000009][info][main] [source/server/server.cc:454] starting main dispatch loop |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Note that I set the –network to “host” in the docker run command. This means that the running container is accessible on localhost without additional network configuration. This works on Linux, but might not work on Windows and Mac. The docker pages suggest you should change the IP address in step #8 of the envoy config to host.docker.internal or gateway.docker.internal according to: [docs.docker.com/docker-for-mac/networking/](https://docs.docker.com/docker-for-mac/networking/#i-want-to-connect-from-a-container-to-a-service-on-the-host)

## **Using your service via HTTP**

If all goes well, you can now cURL your service using HTTP. On Linux, you can connect to localhost, but on windows or mac you might have to connect to the IP address of the VM or docker container. The examples use localhost as there are many ways you can configure docker.

### CREATE A RESERVATION VIA HTTP





| 1234567891011121314151617181920 | curl -X POST http://localhost:51051/v1/reservations \          -H 'Content-Type: application/json' \          -d '{            "title": "Lunchmeeting2",            "venue": "JDriven Coltbaan 3",            "room": "atrium",            "timestamp": "2018-10-10T11:12:13",            "attendees": [                {                    "ssn": "1234567890",                    "firstName": "Jimmy",                    "lastName": "Jones"                },                {                    "ssn": "9999999999",                    "firstName": "Dennis",                    "lastName": "Richie"                }            ]        }' |
| ------------------------------- | ------------------------------------------------------------ |
|                                 |                                                              |

Example output:



| 12345 | {        "id": "2cec91a7-d2d6-4600-8cc3-4ebf5417ac4b",        "title": "Lunchmeeting2",        "venue": "JDriven Coltbaan 3",... |
| ----- | ------------------------------------------------------------ |
|       |                                                              |



### RETRIEVE A RESERVATION VIA HTTP

Use the ID that the POST created!



| 1    | curl http://localhost:51051/v1/reservations/ENTER-ID-HERE! |
| ---- | ---------------------------------------------------------- |
|      |                                                            |

The output should be the same as the result of the create

### LIST RESERVATIONS VIA HTTP

For this example it might be nice to run CreateReservation multiple times with different fields to see the filter in action.



| 1    | curl "http://localhost:51051/v1/reservations" |
| ---- | --------------------------------------------- |
|      |                                               |





| 1    | curl "http://localhost:51051/v1/reservations?room=atrium" |
| ---- | --------------------------------------------------------- |
|      |                                                           |





| 1    | curl "http://localhost:51051/v1/reservations?room=atrium&attendees.lastName=Jones" |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

The response will be an array of Reservations.

### DELETE A RESERVATION





| 1    | curl -X DELETE http://localhost:51051/v1/reservations/ENTER-ID-HERE! |
| ---- | ------------------------------------------------------------ |
|      |                                                              |



# Returned headers

gRPC returns several HTTP headers. Some might might help you with debugging:

- grpc-status: the value is the ordinal value of io.grpc.Status.Code. It can come in handy to see what status gRPC returns.
- grpc-message: the error in case something went wrong.

Visit for more info: [github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)

## **Imperfections**

#### 1. Weird responses if path does not exist.

Envoy does a good job but it sometimes returns, in my opinion, an incorrect status code. For example, when I GET a valid reservation:



| 1    | curl http://localhost:51051/v1/reservations/ENTER-ID-HERE! |
| ---- | ---------------------------------------------------------- |
|      |                                                            |

It returns 200, which is good. But then if I do this



| 1    | curl http://localhost:51051/v1/reservations/ENTER-ID-HERE!/blabla |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Envoy returns:



| 123  | 415 Unsupported Media Type Content-Type is missing from the request |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

I expect 404 here, and the body doesn’t really explain the error well. I filed an issue here: [github.com/envoyproxy/envoy/issues/5010](https://github.com/envoyproxy/envoy/issues/5010)

**RESOLVED**: Envoy was routing all requests to the gRPC service, if the path did not exist in the gRPC service, the gRPC service itself responded with that error. The solution is to make Envoy only forward requests that have an implementation in the gRPC service by adding ‘grpc : {}’ to the configuration of envoy:



| 1234567 | name: local_route            virtual_hosts:            - name: local_service              domains: ["*"]              routes:              - match: { prefix: "/" , grpc: {}}  # <--- this fixes it                route: { cluster: grpc-backend-services, timeout: { seconds: 60 } } |
| ------- | ------------------------------------------------------------ |
|         |                                                              |



#### 2. Sometimes when querying a collection the resource returns ‘[ ]’ even when the server responds with an error.

I filed an issue with the envoy developers [github.com/envoyproxy/envoy/issues/5011](https://github.com/envoyproxy/envoy/issues/5011)

## Upcoming features

In the future it will also be possible to provide subfields of the response message that you want to return in the response body, in case you do not want to return the complete response body. This can be done via a “response_body” field inside the HTTP option. This could be nice if you want to cut out a wrapper object in your HTTP API.

# Final words

I hope this gives a good overview on transcoding a gRPC API to HTTP/JSON.