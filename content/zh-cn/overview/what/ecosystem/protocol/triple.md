---
type: docs
title: "Triple"
linkTitle: "Triple"
weight: 20
description: ""
---

## 概述说明

Triple 协议是 Dubbo3 推出的主力协议。Triple 意为第三代，通过 Dubbo1.0/ Dubbo2.0 两代协议的演进，以及云原生带来的技术标准化浪潮，Dubbo3 新协议 Triple 应运而生。

### RPC 协议的选择

协议是 RPC 的核心，它规范了数据在网络中的传输内容和格式。除必须的请求、响应数据外，通常还会包含额外控制数据，如单次请求的序列化方式、超时时间、压缩方式和鉴权信息等。

协议的内容包含三部分
- 数据交换格式： 定义 RPC 的请求和响应对象在网络传输中的字节流内容，也叫作序列化方式
- 协议结构： 定义包含字段列表和各字段语义以及不同字段的排列方式
- 协议通过定义规则、格式和语义来约定数据如何在网络间传输。一次成功的 RPC 需要通信的两端都能够按照协议约定进行网络字节流的读写和对象转换。如果两端对使用的协议不能达成一致，就会出现鸡同鸭讲，无法满足远程通信的需求。

![协议选择](/imgs/v3/concepts/triple-protocol.png)

RPC 协议的设计需要考虑以下内容：
- 通用性： 统一的二进制格式，跨语言、跨平台、多传输层协议支持
- 扩展性： 协议增加字段、升级、支持用户扩展和附加业务元数据
- 性能：As fast as it can be
- 穿透性：能够被各种终端设备识别和转发：网关、代理服务器等
  通用性和高性能通常无法同时达到，需要协议设计者进行一定的取舍。

#### HTTP/1.1

比于直接构建于 TCP 传输层的私有 RPC 协议，构建于 HTTP 之上的远程调用解决方案会有更好的通用性，如WebServices 或 REST 架构，使用 HTTP + JSON 可以说是一个事实标准的解决方案。

选择构建在 HTTP 之上，有两个最大的优势：

- HTTP 的语义和可扩展性能很好的满足 RPC 调用需求。
- 通用性，HTTP 协议几乎被网络上的所有设备所支持，具有很好的协议穿透性。

但也存在比较明显的问题：

- 典型的 Request – Response 模型，一个链路上一次只能有一个等待的 Request 请求。会产生 HOL。
- Human Readable Headers，使用更通用、更易于人类阅读的头部传输格式，但性能相当差
- 无直接 Server Push 支持，需要使用 Polling Long-Polling 等变通模式

#### gRPC
上面提到了在 HTTP 及 TCP 协议之上构建 RPC 协议各自的优缺点，相比于 Dubbo 构建于 TCP 传输层之上，Google 选择将 gRPC 直接定义在 HTTP/2 协议之上。
gRPC 的优势由HTTP2 和 Protobuf 继承而来。

- 基于 HTTP2 的协议足够简单，用户学习成本低，天然有 server push/ 多路复用 / 流量控制能力
- 基于 Protobuf 的多语言跨平台二进制兼容能力，提供强大的统一跨语言能力
- 基于协议本身的生态比较丰富，k8s/etcd 等组件的天然支持协议，云原生的事实协议标准

但是也存在部分问题

- 对服务治理的支持比较基础，更偏向于基础的 RPC 功能，协议层缺少必要的统一定义，对于用户而言直接用起来并不容易。
- 强绑定 protobuf 的序列化方式，需要较高的学习成本和改造成本，对于现有的偏单语言的用户而言，迁移成本不可忽视

#### 最终的选择 Triple
最终我们选择了兼容 gRPC ，以 HTTP2 作为传输层构建新的协议，也就是 Triple。

容器化应用程序和微服务的兴起促进了针对负载内容优化技术的发展。 客户端中使用的传统通信协议（ RESTFUL或其他基于 HTTP 的自定义协议）难以满足应用在性能、可维护性、扩展性、安全性等方便的需求。一个跨语言、模块化的协议会逐渐成为新的应用开发协议标准。自从 2017 年 gRPC 协议成为 CNCF 的项目后，包括 k8s、etcd 等越来越多的基础设施和业务都开始使用 gRPC 的生态，作为云原生的微服务化框架， Dubbo 的新协议也完美兼容了 gRPC。并且，对于 gRPC 协议中一些不完善的部分， Triple 也将进行增强和补充。

那么，Triple 协议是否解决了上面我们提到的一系列问题呢？

- 性能上: Triple 协议采取了 metadata 和 payload 分离的策略，这样就可以避免中间设备，如网关进行 payload 的解析和反序列化，从而降低响应时间。
- 路由支持上，由于 metadata 支持用户添加自定义 header ，用户可以根据 header 更方便的划分集群或者进行路由，这样发布的时候切流灰度或容灾都有了更高的灵活性。
- 安全性上，支持双向TLS认证（mTLS）等加密传输能力。
- 易用性上，Triple 除了支持原生 gRPC 所推荐的 Protobuf 序列化外，使用通用的方式支持了 Hessian / JSON 等其他序列化，能让用户更方便的升级到 Triple 协议。对原有的 Dubbo 服务而言，修改或增加 Triple 协议 只需要在声明服务的代码块添加一行协议配置即可，改造成本几乎为 0。

### Triple 协议

![Triple 协议通信方式](/imgs/v3/concepts/triple.png)

- 现状

1、完整兼容grpc、客户端/服务端可以与原生grpc客户端打通

2、目前已经经过大规模生产实践验证，达到生产级别

- 特点与优势

1、具备跨语言互通的能力，传统的多语言多 SDK 模式和 Mesh 化跨语言模式都需要一种更通用易扩展的数据传输格式。

2、提供更完善的请求模型，除了 Request/Response 模型，还应该支持 Streaming 和 Bidirectional。

3、易扩展、穿透性高，包括但不限于 Tracing / Monitoring 等支持，也应该能被各层设备识别，网关设施等可以识别数据报文，对 Service Mesh 部署友好，降低用户理解难度。

4、多种序列化方式支持、平滑升级

5、支持 Java 用户无感知升级，不需要定义繁琐的 IDL 文件，仅需要简单的修改协议名便可以轻松升级到 Triple 协议

#### Triple 协议内容介绍

基于 grpc 协议进行进一步扩展

- Service-Version → "tri-service-version" {Dubbo service version}
- Service-Group → "tri-service-group" {Dubbo service group}
- Tracing-ID → "tri-trace-traceid" {tracing id}
- Tracing-RPC-ID → "tri-trace-rpcid" {_span id _}
- Cluster-Info → "tri-unit-info" {cluster infomation}

其中 Service-Version 跟 Service-Group 分别标识了 Dubbo 服务的 version 跟 group 信息，因为grpc的 path 申明了 service name 跟 method name，相比于 Dubbo 协议，缺少了version 跟 group 信息；Tracing-ID、Tracing-RPC-ID 用于全链路追踪能力，分别表示 tracing id 跟 span id 信息；Cluster-Info 表示集群信息，可以使用其构建一些如集群划分等路由相关的灵活的服务治理能力。

#### Triple Streaming

Triple协议相比传统的unary方式，多了目前提供的Streaming RPC的能力

- Streaming 用于什么场景呢？

在一些大文件传输、直播等应用场景中， consumer或provider需要跟对端进行大量数据的传输，由于这些情况下的数据量是非常大的，因此是没有办法可以在一个RPC的数据包中进行传输，因此对于这些数据包我们需要对数据包进行分片之后，通过多次RPC调用进行传输，如果我们对这些已经拆分了的RPC数据包进行并行传输，那么到对端后相关的数据包是无序的，需要对接收到的数据进行排序拼接，相关的逻辑会非常复杂。但如果我们对拆分了的RPC数据包进行串行传输，那么对应的网络传输RTT与数据处理的时延会是非常大的。

为了解决以上的问题，并且为了大量数据的传输以流水线方式在consumer与provider之间传输，因此Streaming RPC的模型应运而生。

通过Triple协议的Streaming RPC方式，会在consumer跟provider之间建立多条用户态的长连接，Stream。同一个TCP连接之上能同时存在多个Stream，其中每条Stream都有StreamId进行标识，对于一条Stream上的数据包会以顺序方式读写。

## 使用方式 - Java

### Protobuf 方式

1. 编写 IDL 文件
    ```protobuf
    syntax = "proto3";
    option java_multiple_files = true;
    option java_package = "org.apache.dubbo.hello";
    option java_outer_classname = "HelloWorldProto";
    option objc_class_prefix = "HLW";
    package helloworld;
    // The request message containing the user's name.
    message HelloRequest {
      string name = 1;
    }
    // The response message containing the greetings
    message HelloReply {
      string message = 1;
    }
    ```

2. 添加编译 protobuf 的 extension 和 plugin (以 maven 为例)
    ```xml
       <extensions>
                <extension>
                    <groupId>kr.motd.maven</groupId>
                    <artifactId>os-maven-plugin</artifactId>
                    <version>1.6.1</version>
                </extension>
            </extensions>
            <plugins>
                <plugin>
                    <groupId>org.xolstice.maven.plugins</groupId>
                    <artifactId>protobuf-maven-plugin</artifactId>
                    <version>0.6.1</version>
                    <configuration>
                        <protocArtifact>com.google.protobuf:protoc:3.7.1:exe:${os.detected.classifier}</protocArtifact>
                        <pluginId>triple-java</pluginId>
                        <outputDirectory>build/generated/source/proto/main/java</outputDirectory>
                    </configuration>
                    <executions>
                        <execution>
                            <goals>
                                <goal>compile</goal>
                                <goal>test-compile</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
    ```

3. 构建/ 编译生成 protobuf Message 类
    ```shell
    mvn clean install
    ```

### Unary 方式

1.  编写 Java 接口
    ```java
    import org.apache.dubbo.hello.HelloReply;
    import org.apache.dubbo.hello.HelloRequest;
    public interface IGreeter {
        /**
         * <pre>
         *  Sends a greeting
         * </pre>
         */
        HelloReply sayHello(HelloRequest request);
    }
    ```

2. 创建 Provider
    ```java
        public static void main(String[] args) throws InterruptedException {
            ServiceConfig<IGreeter> service = new ServiceConfig<>();
            service.setInterface(IGreeter.class);
            service.setRef(new IGreeter1Impl());
            // 这里需要显示声明使用的协议为triple 
            service.setProtocol(new ProtocolConfig(CommonConstants.TRIPLE, 50051));
            service.setApplication(new ApplicationConfig("demo-provider"));
            service.setRegistry(new RegistryConfig("zookeeper://127.0.0.1:2181"));
            service.export();
            System.out.println("dubbo service started");
            new CountDownLatch(1).await();
        }
    ```


3. 创建 Consumer

    ```java
    public static void main(String[] args) throws IOException {
        ReferenceConfig<IGreeter> ref = new ReferenceConfig<>();
        ref.setInterface(IGreeter.class);
        ref.setCheck(false);
        ref.setProtocol(CommonConstants.TRIPLE);
        ref.setLazy(true);
        ref.setTimeout(100000);
        ref.setApplication(new ApplicationConfig("demo-consumer"));
        ref.setRegistry(new RegistryConfig("zookeeper://127.0.0.1:2181"));
        final IGreeter iGreeter = ref.get();
        System.out.println("dubbo ref started");
        try {
            final HelloReply reply = iGreeter.sayHello(HelloRequest.newBuilder()
                    .setName("name")
                    .build());
            TimeUnit.SECONDS.sleep(1);
            System.out.println("Reply:" + reply);
        } catch (Throwable t) {
            t.printStackTrace();
        }
        System.in.read();
    }
    ```

4. 运行 Provider 和 Consumer ,可以看到请求正常返回
    ```java
   > Reply:message: "name"
    ```

### stream 方式

1.  编写 Java 接口
    ```java
    import org.apache.dubbo.hello.HelloReply;
    import org.apache.dubbo.hello.HelloRequest;
    public interface IGreeter {
        /**
     	* <pre>
     	*  Sends greeting by stream
     	* </pre>
    	 */
	    StreamObserver<HelloRequest> sayHello(StreamObserver<HelloReply> replyObserver);
    }
    ```

2. 编写实现类
    ```java
	public class IStreamGreeterImpl implements IStreamGreeter {
	    @Override
	    public StreamObserver<HelloRequest> sayHello(StreamObserver<HelloReply> replyObserver) {
	        return new StreamObserver<HelloRequest>() {
	            private List<HelloReply> replyList = new ArrayList<>();
	            @Override
	            public void onNext(HelloRequest helloRequest) {
	                System.out.println("onNext receive request name:" + helloRequest.getName());
	                replyList.add(HelloReply.newBuilder()
	                    .setMessage("receive name:" + helloRequest.getName())
	                    .build());
	            }
	            @Override
	            public void onError(Throwable cause) {
	                System.out.println("onError");
	                replyObserver.onError(cause);
	            }
	            @Override
	            public void onCompleted() {
	                System.out.println("onComplete receive request size:" + replyList.size());
	                for (HelloReply reply : replyList) {
	                    replyObserver.onNext(reply);
	                }
	                replyObserver.onCompleted();
	            }
	        };
	    }
	}
    ```

3. 创建 Provider

    ```java
	public class StreamProvider {
	    public static void main(String[] args) throws InterruptedException {
	        ServiceConfig<IStreamGreeter> service = new ServiceConfig<>();
	        service.setInterface(IStreamGreeter.class);
	        service.setRef(new IStreamGreeterImpl());
	        service.setProtocol(new ProtocolConfig(CommonConstants.TRIPLE, 50051));
	        service.setApplication(new ApplicationConfig("stream-provider"));
	        service.setRegistry(new RegistryConfig("zookeeper://127.0.0.1:2181"));
	        service.export();
	        System.out.println("dubbo service started");
	        new CountDownLatch(1).await();
	    }
	}
    ```

4. 创建 Consumer

    ```java
	public class StreamConsumer {
	    public static void main(String[] args) throws InterruptedException, IOException {
	        ReferenceConfig<IStreamGreeter> ref = new ReferenceConfig<>();
	        ref.setInterface(IStreamGreeter.class);
	        ref.setCheck(false);
	        ref.setProtocol(CommonConstants.TRIPLE);
	        ref.setLazy(true);
	        ref.setTimeout(100000);
	        ref.setApplication(new ApplicationConfig("stream-consumer"));
	        ref.setRegistry(new RegistryConfig("zookeeper://mse-6e9fda00-p.zk.mse.aliyuncs.com:2181"));
	        final IStreamGreeter iStreamGreeter = ref.get();
	        System.out.println("dubbo ref started");
	        try {
	            StreamObserver<HelloRequest> streamObserver = iStreamGreeter.sayHello(new StreamObserver<HelloReply>() {
	                @Override
	                public void onNext(HelloReply reply) {
	                    System.out.println("onNext");
	                    System.out.println(reply.getMessage());
	                }
	                @Override
	                public void onError(Throwable throwable) {
	                    System.out.println("onError:" + throwable.getMessage());
	                }
	                @Override
	                public void onCompleted() {
	                    System.out.println("onCompleted");
	                }
	            });
	            streamObserver.onNext(HelloRequest.newBuilder()
	                .setName("tony")
	                .build());
	            streamObserver.onNext(HelloRequest.newBuilder()
	                .setName("nick")
	                .build());
	            streamObserver.onCompleted();
	        } catch (Throwable t) {
	            t.printStackTrace();
	        }
	        System.in.read();
	    }
	}
    ```

5. 运行 Provider 和 Consumer ,可以看到请求正常返回了
    ```java
    > onNext\
    > receive name:tony\
    > onNext\
    > receive name:nick\
    > onCompleted
    ```

### 其他序列化方式
省略上文中的 1-3 步，指定 Provider 和 Consumer 使用的协议即可完成协议升级。

## 使用方式 - Go

TBD

## 使用方式 - Node.js

暂不支持

## 使用方式 - Rust

TBD

## 常见问题

### Q1
protobuf 类找不到

由于 Triple 协议底层需要依赖 protobuf 协议进行传输，即使定义的服务接口不使用 protobuf 也需要在环境中引入 protobuf 的依赖。

```xml
        <dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>3.19.4</version>
        </dependency>
```

### Q2
如何原生接入网关？

对于需要网关接入的 Dubbo 用户，Triple 协议提供了更加原生的方式，让网关开发或者使用开源的 grpc 网关组件更加简单。网关可以选择不解析 payload ，在性能上也有很大提高。在使用 Dubbo 协议时，语言相关的序列化方式是网关的一个很大痛点，而传统的 HTTP 转 Dubbo 的方式对于跨语言序列化几乎是无能为力的。同时，由于 Triple 的协议元数据都存储在请求头中，网关可以轻松的实现定制需求，如路由和限流等功能。

### 总结

在API领域，最重要的趋势是标准化技术的崛起。Triple 协议是 Dubbo3 推出的主力协议。它采用分层设计，其数据交换格式基于Protobuf (Protocol Buffers) 协议开发，具备优秀的序列化/反序列化效率，当然还支持多种序列化方式，也支持众多开发语言。在传输层协议，Triple 选择了 HTTP/2，相较于 HTTP/1.1，其传输效率有了很大提升。此外HTTP/2作为一个成熟的开放标准，具备丰富的安全、流控等能力，同时拥有良好的互操作性。Triple 不仅可以用于Server端服务调用，也可以支持浏览器、移动App和IoT设备与后端服务的交互，同时 Triple协议无缝支持 Dubbo3 的全部服务治理能力。

在Cloud Native的潮流下，跨平台、跨厂商、跨环境的系统间互操作性的需求必然会催生基于开放标准的RPC技术，而gRPC顺应了历史趋势，得到了越来越广泛地应用。在微服务领域，Triple协议的提出与落地，是 Dubbo3 迈向云原生微服务的一大步。
