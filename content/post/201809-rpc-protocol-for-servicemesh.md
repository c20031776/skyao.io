+++
title = "转发的艺术：ServiceMesh下的RPC通讯协议"

date = 2018-09-12
lastmod = 2018-09-12
draft = true

tags = ["RPC", "ServiceMesh"]
summary = "ServiceMesh下的RPC通讯协议"

abstract = "ServiceMesh下的RPC通讯协议"

[header]
image = "headers/post/201809-xprotocol-in-sofamesh.jpg"
caption = ""

+++



## 背景介绍

在2018年上半年，蚂蚁金服决定基于Istio订制自己的ServiceMesh解决方案，在6月底对外公布了SOFAMesh，详情请见下文:

- [大规模微服务架构下的Service Mesh探索之路](../../publication/service-mesh-explore/)

在SOFAMesh的开发过程中，为了解决我们遇到的多通讯协议支持，同时尽量提升性能，我们给出了一套名为X-Protocol的解决方案。今天我们将对此进行详细的讲解。

### 不一样的需求：协议扩展

在Istio和Envoy中，对通讯协议的支持，主要体现在HTTP/1.1和HTTP/2上，这两个是Istio/Envoy中的一等公民。而基于HTTP/1.1的REST和基于HTTP/2的gRPC，一个是目前社区最主流的通讯协议，一个是未来的主流，google的宠儿，CNCF御用的RPC方案，这两个组成了目前Istio和Envoy（乃至CNCF所有项目）的黄金组合。

而我们SOFAMesh，在第一时间就遇到和Istio/Envoy不同的情况，我们需要支持REST和gRPC之外的众多协议：

- SOFARPC：这是蚂蚁金服大量使用的RPC协议(已开源)
- HSF RPC：这是阿里集团内部大量使用的RPC协议(未开源)
- Dubbo RPC: 这是社区广泛使用的RPC协议(已开源)
- 其他私有协议：在过去几个月间，我们收到需求，期望在SOFAMesh上运行其他TCP协议，部分是私有协议

为此，我们需要考虑在SOFAMesh和SOFAMOSN中增加这些通讯协议的支持，尤其是要可以方便的扩展支持各种私有TCP协议：

![](images/protocol-supported.jpg)

### 更适合的平衡点：性能和功能

对于服务间通讯解决方案，性能永远是一个值得关注的点。而SOFAMesh在项目启动时就明确要求在性能上要有更高的追求，为此，我们不得不在Istio标准实现之外寻求可以获取更高性能的方式。

期间有两个发现：

1. Istio在处理所有的请求转发如REST/gRPC时，会解码整个请求的header信息，拿到各种数据，提取为Attribute，然后以此为基础，提供各种丰富的功能，典型如Content Based Routing。

2. 而在测试中，我们发现：解码请求协议的header部分，对CPU消耗较大，直接影响性能。

因此，我们有了一个很简单的想法：是不是可以在转发时，不开启部分功能，以此换取转发过程中的更少更快的解码，甚至不解码。毕竟，不是每个服务都需要用到那么多的高级特性，而对于部分对性能有极高追求的服务，不开启额外功能而换取最高的性能，也是一种折中方案。

## 请求转发

在我们进一步深入前，我们先来探讨一下实现请求转发的技术细节。

有一个关键问题：当Envoy/MOSN这样的代理程序，接收到来自客户端的TCP请求时，需要获得哪些信息，才可以正确的转发请求到上游的服务器端？

### 最关键的信息：destination

首先，毫无疑问的，必须拿到destination/目的地，也就是客户端请求必须通过某种方式明确的告之代理该请求的destination，这样代理程序才能根据这个destionation去找到正确的目标服务器，然后才有后续的连接目标服务器和转发请求等操作。

![](images/destination-in-forward.jpg)

Destination信息的表述形式可能有：

1. IP地址

	可能是服务器端实例实际工作的IP地址和端口，也可能是某种转发机制，如Nginx/HAProxy等反向代理的地址，或者k8s中的ClusterIP。

	举例："192.168.1.1:8080"是实际IP地址和端口，"10.2.0.100:80"是ngxin反向代理地址，"172.168.1.105:80"是k8s的ClusterIP。

2. 目标服务的标识符

	可用于名字查找，如服务名，可能带有各种前缀后缀。然后通过名字查找/服务发现等方式，得到地址列表（通常是IP地址+端口形式）。

	举例："userservice"是标准服务名， "com.alipay/userservice"是加了域名前缀的服务名， "service.default.svc.cluster.local"是k8s下完整的全限定名。

Destination信息在请求报文中的携带方式有：

1. 通过通讯协议传递

	这是最常见的形式，标准做法是通过header头，典型如HTTP/1.1下一般使用 host header，举例如"Host: userservice"。HTTP/2下，类似的使用":authority" header。

	对于非HTTP协议，通常也会有类似的设计，通过协议中某些字段来承载目标地址信息，只是不同协议中这个字段的名字各有不同。如SOFARPC，HSF等。

	有些通讯协议，可能会将这个信息存放在payload中，比如后面我们会介绍到的dubbo协议，导致需要反序列化payload之后才能拿到这个重要信息，对转发性能影响很大。

2. 通过TCP协议传递

	这是一种非常特殊的方式，通过在TCP option传递，后面讲Istio DNS寻址时会详细介绍。
	
### TCP拆包

如何从请求的通讯协议中获取destination？这涉及到具体通讯协议的解码，其中第一个要解决的问题就是如何在连续的TCP报文中将每个请求内容拆分开，这里就涉及到经典的TCP沾包、拆包问题。

![](images/tcp-packge.jpg)

由于篇幅和主题限制，我们不在这里展开TCP沾包、拆包的原理。后面针对每个具体的通讯协议进行分析时再具体看各个协议的解决方案。

### 多路复用的关键参数：RequestId

RequestId用来关联request和对应的response，请求报文中携带一个唯一的id值，应答报文中原值返回，以便在处理response时可以找到对应的request。当然在不同协议中，这个参数的名字可能不同。

严格说，RequestId对于请求转发是可选的，也有很多通讯协议不提供支持，比如经典的HTTP1.1就没有支持。但是如果有这个参数，则可以实现多路复用，从而可以大幅度提高TCP连接的使用效率，避免出现大量连接。稍微新一点的通讯协议，基本都会原生支持这个特性，比如SOFARPC，dubbo，HSF，还有HTTP/2就直接內建了多路复用的支持。

HTTP/1.1不支持多路复用，用的是经典的ping-pong模式：在请求发送之后，必须独占当前连接，等待服务器端给出这个请求的应答，然后才能释放连接。因此HTTP/1.1下，并发多个请求就必须采用多连接，为了提升性能通常会使用长连接+连接池的设计。而如果有了requestid和多路复用的支持，客户端和Mesh之间就可以只用一条连接支持并发请求：

![](images/requestid-1.jpg)

而Mesh与服务器（也可能是对端的Mesh）之间，也同样可以受益于多路复用技术，来自不同客户端而去往同一个目的地的请求可以混杂在同一条连接上发送。通过RequestId的关联，Mesh可以正确将reponse发送到请求来自的客户端。

![](images/requestid-2.jpg)

由于篇幅和主题限制，我们不在这里展开多路复用的原理。后面针对每个具体的通讯协议进行分析时再具体看各个协议的支持情况。

## 主流通讯协议

现在我们开始，以Proxy、Sidecar、Service Mesh的角度来看看目前主流的通讯协议和我们前面列举的需要在SOFAMesh中支持的几个协议。

### SOFARPC/bolt协议

SOFARPC 是一款基于 Java 实现的 RPC 服务框架，详细资料可以查阅 [官方文档](http://www.sofastack.tech/sofa-rpc/docs/Home)。SOFARPC 支持 [bolt](https://github.com/alipay/sofa-bolt)，rest，dubbo 协议进行通信。REST、dubbo后面单独展开，这里我们关注bolt协议。

bolt 是蚂蚁金融服务集团开放的基于 Netty 开发的网络通信框架，其协议格式是变长，即协议头+payload。具体格式定义如下，以request为例（response类似）：

![](images/bolt-protocol-request.jpg)

我们只关注和请求转发直接相关的字段：

- TCP拆包

	botl协议是定长+变长的复合结构，前面22个字节长度固定，每个字节和协议字段的对应如图所示。其中classLen，headerLen和contentLen三个字段指出后面三个变长字段className，header，content的实际长度。和通常的变长方案相比只是变长字段有三个。拆包时思路简单明了：

	1. 先读取前22个字节，解出各个协议字段的实际值，包括classLen，headerLen和contentLen
	2. 按照classLen，headerLen和contentLen的大小，继续读取className，header，content

- Destination

	Bolt协议中的header字段是一个map，其中有一个key为"service"的字段，传递的是接口名/服务名。读取稍微麻烦一点点，需要先解开header字段。

- RequestId

	Blot协议固定字段中的`requestID`字段，可以直接读取。

总结：SOFARPC中的bolt协议，设计的非常符合请求转发的需要，TCP拆包，读取RequestID和Destination，可谓一气呵成。除了Destination的获取需要解码整个header外，近乎完美。

### HSF协议

HSF协议是经过精心设计工作在4层的私有协议，由于该协议没有开源，因此不便直接暴露具体格式和字段详细定义。

不过基本的设计和bolt非常类似：

- 采用变长格式，即协议头+payload
- 在协议头中可以直接拿到服务接口名和服务方法名作为Destination
- 有RequestID字段

总结：基本和bolt一致，考虑到Destination可以直接读取，比bolt还要方便一些，HSF协议可以说是对请求转发最完美的协议。

### Dubbo协议

Dubbo协议也是类似的协议头+payload的变长结构，其协议格式如下：

![](images/dubbo-protocol.jpg)

其中long类型的`id`字段用来把请求request和返回的response对应上，即我们所说的`RequestId`。

这样TCP拆包和多路复用都轻松实现，稍微麻烦一点的是：Destination在哪里？Dubbo在这里的设计有点不够理想，在协议头中没有字段可以直接读取到Destination，需要去读取data字段，也就是payload，里面的path字段通常用来保存服务名或者接口名。

以hession2为例，data字段的组合是：dubbo version + path + interface version + method + ParameterTypes + Arguments + Attachments。每个字段都是一个byte的长度+字段值的UTF bytes。因此读取时也不复杂，速度也足够快。

总结：基本和HSF一致，就是Destination的读取稍微麻烦一点，整体说还是很适合转发的。

### HTTP/1.1

HTTP/1.1的格式应该大家都熟悉，而在这里，不得不指出，HTTP/1.1协议对请求转发是非常不友好的：

1. HTTP请求在拆包时，需要先按照HTTP header的格式，一行一行读取，直到出现空行表示header结束
2. 然后必须将整个header的内容全部解析出来，才能取出`Content-Length header`
3. 通过`Content-Length` 值，才能完成对body内容的读取，实现正确拆包
4. 如果是chunked方式，则更复杂一些
5. Destination通常从`Host` header中获取

这意味着，为了完成最基本的TCP拆包，必须完整的解析全部的HTTP header信息，没有任何可以优化的空间。对比上面几个RPC协议，轻松自如的快速获取几个关键信息，HTTP无疑要重很多。

### HTTP/2和gRPC

作为HTTP/1.1的接班人，HTTP/2则表现的要好很多。

> 备注：当然HTTP/2的协议格式复杂多了，由于篇幅和主题的限制，这里不详细介绍HTTP/2的格式。

首先HTTP/2是以帧的方式组织报文的，所有的帧都是变长，固定的9个字节+可变的payload，Length字段指定payload的大小：

![](images/http2-frame.png)

HTTP2的请求和应答，也被称为Message，是由多个帧构成，在去除控制帧之外，Message通常由Header帧开始，后面接CONTINUATION帧和Data帧（也可能没有，如GET请求）。每个帧都可以通过头部的Flags字段来设置END_STREAM标志，表示请求或者应答的结束。即TCP拆包的问题在HTTP/2下是有非常标准而统一的方式完成，完全和HTTP/2上承载的协议无关。

HTTP/2通过Stream內建多路复用，这里的`Stream Identifier` 扮演了类似前面的`RequestId`的角色。

而Destination信息则通过Header帧中的伪header `:authority` 来传递，类似HTTP/1.1中的`Host` header。不过HTTP/2下header会进行压缩，读取时稍微复杂一点。

gRPC协议基于HTTP/2，自然继承了HTTP/2的上述优点。

总结：HTTP/2的先进设计，在请求转发时表现的非常友好。

