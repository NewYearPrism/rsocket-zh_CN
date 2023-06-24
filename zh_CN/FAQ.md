### 常见问题

#### 为什么要搞一个新的协议？

关于动机的完整说明请看 [Motivations.md](./Motivations.md)。

其中包括几个重要的理由:

- 除了请求/响应(request/response)模型之外更多的交互模型，例如流式响应和推送(streaming responses and push)
- 在不同的网络边界上(across network boundaries)的应用层流量控制语义 (异步的有限大小的批处理拉取/推送(async pull/push of bounded batch sizes))
- 对单一连接进行二进制(binary)多路复用（译者留言：不是很明白此处binary的含义）
- 在不同的传输层连接上(across transport connections)支持长时间订阅的可恢复性(resumption)
- 需要一个应用层协议来使用各种传输层协议，例如 WebSockets 和 [Aeron]: https://github.com/real-logic/Aeron （译者留言：原文就是这么写的。。）

#### 为什么不用 某某技术 将就一下呢？

归根到底，只要投入足够的精力，以上的目的用任何技术都可以达成。但是，创立本项目的相关人士追求一套更清晰和更正式的体系——一个优美的解决方案，而不是“将就一下”。

#### 为什么不用 HTTP/2？

虽然 HTTP/2 **远**比 HTTP/1 更适合浏览器和请求/响应模式的文档传输，但是它并没有提供除请求/响应之外的交互模型，也不支持应用层流量控制。

从 HTTP/2规范 和 常见问题 摘录了一些文字，以方便大家了解 HTTP/2 针对的方向:

> “HTTP's existing semantics remain unchanged.” HTTP已存在的语义没有改变

> “… from the application perspective, the features of the protocol are largely unchanged …” 在应用的角度，协议的特性很大程度上没有改变

> "This effort was chartered to work on a revision of the wire protocol – i.e., how HTTP headers, methods, etc. are put “onto the wire”, not change HTTP’s semantics." （略）

Additionally, "push promises" are focused on filling browser caches for standard web browsing behavior: （译者留言：不懂）

> “Pushed responses are always associated with an explicit request from the client.”

这说明我们仍然需要SSE或者WebSockets来进行推送操作（SSE是一个文本协议，要求UTF-8编码的Base64）。

HTTP/2 作为 “更好的 HTTP/1.1”，主要用于浏览器从网站检索文档。对于应用程序，RSocket 可以做得更好。

欲了解更多请阅读 RSocket [动机文档](./Motivations.md)。

#### 那 QUIC 呢？

QUIC 生态目前还比较荒芜。（译者注：这段应该是2015年左右写的。）如果 QUIC 有所发展，我们希望可以将 QUIC 用作 RSocket 的传输层。

其实 RSocket 就是计划专门建立在像 QUIC 这样的传输层之上的。QUIC 提供了可靠的传输(reliable transport)，拥塞控制(congestion control)，字节级别的流量控制(byte-level flow control)，和多路复用字节流(multiplexed byte streams)。RSocket 在此之上建立各种机制，包括二进制帧封装(binary framing)和消息流（单向和双向）(message streams (unidirectional and bidirectional))，消息级别的流量控制(message-level flow control)，可恢复性(resumption)等行为语义。

RSocket 规范是在考虑了分层的情况下制定的，因此，在类似 TCP 的协议上，RSocket 会包含帧长度(frame length)和流标识(stream ID)。但如果在 HTTP/2 或 QUIC 上，RSocket 就可以直接使用 HTTP/2 或 QUIC 提供的相关机制而不需要自己处理。

请看 ["帧封装格式"](./Protocol.md):

> When using a transport protocol that does not provide compatible framing, the Frame Length MUST be prepended to the RSocket Frame. 如果使用的传输协议没有提供兼容的帧封装机制，则 RSokcet 帧**必须**(MUST)在前部附加帧长度数据。

以及 ["帧标头格式"](./Protocol.md): …

> Transport protocols that include demultiplexing, such as HTTP/2, MAY omit the Stream ID field if all parties agree. The means of negotiation and agreement is left to the transport protocol. 像 HTTP/2 这样支持多路分解的传输协议，在双方同意的情况下，**可以**(MAY)省略流标识字段。

#### 为什么要使用 反应式流("Reactive Streams") `request(n)` 流量控制？

如果应用程序不反馈负载单元（不是指一段字节）是否已经处理完毕的信息，就容易造成“队头阻塞(head of line blocking)”，导致应用程序和网络的缓冲区爆满，服务器生产的数据超出客户端的处理能力范围。尤其是在复用同一个连接建立多条数据流时，情况更加严重，因为某一条数据流就可能导致其他所有的流饥饿。应用层面的 `request(n)` 语义让消费者可以指示某条流的接收能力，生产者也能把多条流交织在一起了。

举个例子，如果在使用 TCP 时仅仅只依赖 TCP 本身的流控，那么我们可能会遇到以下问题：

- 在发送端和接收端都有数据缓冲机制，导致根本无法得知订阅者是否已经完成任务
- 发送端在发送超大负载单元时（大于发送端和接收端的缓冲能力），会陷入一种性能低下的行为——TCP 连接会在满载和空载之间来回变化，大幅降低性能和吞吐量。
- TCP 处理单独的一对发送端/接收端，而反应式流(Reactive Stream)不仅可以有多个发送端和接收端，而且解耦了应用消费控制与传输层数据接收。比如某个应用可能希人为地减缓或限制处理速度，而不干预传输层面的数据接收。

本质上这是因为 TCP 和 反应式流二者的流控设计理念是不一样的（TCP：不超过接收端系统缓冲或网络队列的能力范围。 反应式流：建立应用推送/拉取负载单元的语义，更多的传播模型，以及应用指示其是否有能力接收更多数据的能力）。二者着重点的泾渭分明对任何实际系统的高效运转来说很有必要。

这也解释了为什么一个没有在应用层提供流控机制的技术(solution)使用起来不会特别方便（除了MQTT、AMQP和STOMP之外，几乎所有提到的解决方案），和为什么 RSocket 要把应用层流量控制纳入第一等级的需求。

#### 建立连接的需求

事实上和 HTTP/2 的需求一致——为了交换 SETTINGS 帧:

- <https://http2.github.io/http2-spec/#ConnectionHeader>
- <https://http2.github.io/http2-spec/#discover-http>

HTTP/2 和 RSocket 都要求一个有初始化数据交换和有状态的连接。

#### 传输层

HTTP/2 [要求 TCP](https://http2.github.io/http2-spec/#starting). RSocket [要求 TCP, WebSockets 或 Aeron](./Protocol.md#terminology).

我们并不打算让 RSocket 运行在 HTTP/1.1 之上。HTTP/2 同理，因为 HTTP/2 也只是 HTTP/1.1 接口的包装（根据各浏览器中展现的接口），尽管理论上可行（使用 SSE 或块编码）。如果使用的是一个展现了底层字节流的 HTTP/2 实现，则 HTTP/2 也不失为一种可用的传输手段（事实上已经有 RSocket 用户实现了）。（译者按：目前 [rsocket-py](https://github.com/rsocket/rsocket-py) 已经实现了 QUIC 和 HTTP/3 传输）

#### 代理

HTTP/2 可用的代理，RSocket 也可用。 

#### 帧长度

是否需要取决于传输手段。

在 TCP 上传输时需要. 在 Aeron 或 WebSocket 上传输时不需要. 

#### 跨连接的状态(State)

我们认定在本协议的层面这是非必需品，因为它的正常运行肯定会涉及到应用本身。应用本身可以在多个连接上维护状态。它可能导致实现非常复杂，带来的收益却十分有限。很多分布式系统实现[对处理此类问题的尝试都失败了](https://aphyr.com/tags/jepsen)

但是，RSocket 协议还是提供了必要的通信机制，以便客户端和服务端维护状态以及在新的传输层连接上重建会话。

#### 时间的考验

没有什么能够完全经得起时间的考验，但是我们也为 RSocket 的面向未来做了一些尝试：

- 帧类型有一个保留值用于扩展
- 错误码有一个保留值用于扩展
- 连接建立过程有一个版本字段
- 所有字段的大小的安排都是依据当前已知的需求（比如 streamId 就支持40亿个请求(4b requests)）
- 有大量空间供额外的标志位使用
- 分离了数据和元数据(metadata)
- 在连接建立时使用 MimeType 以消除与编码的耦合

而且我们坚持使用 HTTP/2 和 TCP 的面向连接语义，这样连接行为就不会有异常或特殊性。

除这些因素外，TCP 自1977年以来便存在至今，我们认为它短时间内还不会面临淘汰。QUIC 看起来会在这几年间成为 TCP 的正统替代品。既然 HTTP/2 已经可以在 QUIC 上工作，RSocket 肯定也可以。

#### 优先级(Prioritization)，QoS，OOB

优先级，QoS，OOB 可以通过元数据(metadata)，应用逻辑或应用级分发控制等机制实现。RSocket 不强制规定任何队列模型，分发模型或处理模型。为了让 QoS 发挥作用，需要控制其中的各个方面。无需与应用层和底层网络协作是不太现实的。出于同样原因，HTTP/2 也没有涉及这一块领域，仅仅提供了一个表达此类意向的手段。由于有元数据机制，RSocket 甚至无需这种手段了。

#### 为什么要求取消(cancellation)机制？

现代的分布式系统的拓扑结构倾向于拥有多层级的请求输出端，意味着某个层级的一个请求可能会触发数十个对多个不同后端的请求。如果请求可以取消，便有机会节省大量的资源。

#### 有没有取消(cancellation)的使用例？

假设有一个服务器，它的功能是计算π的第n位数。一个客户端向该服务器发送了一个请求，然而随即发现它根本不需要响应结果了（管他什么原因）。幸好客户端可以取消这个请求，而不是让服务器白白浪费算力（服务器甚至可能此时还没有开始计算）。

#### 有没有请求-流(request-stream)的使用例？

假设有一个聊天服务器，你想要接收服务器上的所有聊天消息，但是你并不想轮询或连续轮询（长轮询技术）。另一个例子是我们只想监听某个聊天室并忽略其他一切消息。

#### 有没有使用即发即弃(fire-and-forget)而非请求-响应(request-response)的例子？

某些请求不需要响应，而如果进一步地根本不需要得知是否发送成功，则即发即弃是一个很好的方案。例子是在不适宜使用 UDP 的环境中持续追踪某些非关键指标。

#### 为什么用二进制？

可以看看这里：<https://http2.github.io/faq/#why-is-http2-binary>

#### 二进制编码不是让调试难度更大了吗？

确实，但这是值得的。

二进制编码使消息更难被人类阅读，但同时也降低了机器阅读消息的难度。由于无需对内容进行解码，性能也显著提升了。我们估计超过 99.9999999% 的消息是被机器读取的，所以决定让消息更有利于机器阅读。

现在已经有用于分析二进制协议通信的工具。我们可以轻而易举地编写新工具和扩展来解码二进制 RSocket 格式并展示人类可读的文本。

#### 有现成的 RSocket 调试工具吗？

推荐 Wireshark。插件在这里：https://github.com/rsocket/rsocket-wireshark

#### 为什么需要这些高于传输层的流量控制方式？

TCP 流量控制的目的是通过远程端的消费速率来控制发送端/接收端的字节速率。使用 RSocket 时，我们会复用同一 TCP 连接建立多条数据流，所以 RSocket 协议层必须要拥有自己的流量控制。

#### What are example use cases where RSocket flow control helps?

Flow control helps an application signal its capability to consume responses. This ensures that we never overflow any queue on the application layer.
Relying on the TCP flow control doesn't work, because we multiplex the streams on the same connection.

#### How does RSocket flow control behave?

There are two types of flow control: 

- One is provided by the request-n semantics defined in Reactive Streams (please [read the spec][Reactive Streams] for exhaustive details).
- The second is provided via the lease semantics defined in the [Protocol document](https://github.com/rsocket/rsocket/blob/master/Protocol.md#lease-semantics).

#### How does RSocket benefit a client-side load balancer in a data center?

Each RSocket provides an availability number abstractly representing its capacity to send traffic.
For instance, when a client doesn't have a valid lease, it exposes a "0.0" availability indicating that it can't send any traffic. This extra piece of information, in combination with any load balancing strategy already used, give more information to the client to make smarter decisions.

#### Why is multiplexing more efficient?

- <https://http2.github.io/faq/#why-is-http2-multiplexed>
- <https://http2.github.io/faq/#why-just-one-tcp-connection>

#### Is multiplexing equivalent to pipelining?

No, pipelining requires reading the responses in the order of the requests. 

For example, with pipelining: a client sends `reqA`, `reqB`, `reqC`. It has to receive the responses in this order: `respA`, `respB`, `respC`.

With multiplexing, the same client can receive responses in any order, e.g. `respB`, `respA`, `respC`.

Pipelining can introduce [head-of-line-blocking](https://en.wikipedia.org/wiki/Head-of-line_blocking) and degrade the performance of the system.

#### Why is the "TLS False start" strategy useful for establishing a connection?

When respecting the lease semantics, establishing a RSocket between a client and a server require one round-trip (-> SETUP, <- LEASE, -> REQUEST). On slow network or when the connection latency is important, this round-trip is harmful. That's why you have the possibility to not respect the lease, and then can send your request right away (-> SETUP, -> REQUEST).

#### What are example use cases for payload data on the Setup frame?

You may want to pass data to your application at RSocket establishment, rather than reimplementing a connect protocol on top of RSocket, RSocket provides you the possibility to send information alongside the SETUP frame.
For instance, this can be used by a client to send its credentials.

#### Why multiple interaction models?

The interaction models could be reduced to just one "request-channel". Every other interaction model is a subtype of request-channel, but they have been specialized for two reasons:

- Ease of use from the client point of view.
- Performance.

#### So why the "RSocket" name? 

It started out as ReactiveSocket, but was shortened to RSocket:

- because it is shorter to write and speak
- to stop overusing the word "reactive" 

That said, the "R" still refers to "reactive" from "ReactiveSocket", so ... isn't "Reactive" a totally [hyped](http://www.gartner.com/technology/research/methodologies/hype-cycle.jsp) buzzword? 

Unfortunately the word has become quite a buzzword, and overused.

However, this library is directly related to several projects where "Reactive" is an important part of their name and architectural pattern. Specifically, RSocket implements, uses, or follows the principles in these projects and libraries, thus the name. 

[Reactive Streams]: http://www.reactive-streams.org
[Reactive Extensions]: http://www.reactivex.io
[RxJava]: https://github.com/ReactiveX/RxJava
[RxJS]: https://github.com/ReactiveX/RxJS
[Reactive Manifesto]: http://www.reactivemanifesto.org
