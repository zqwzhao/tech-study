

Netty面试题总结
======

### Netty 线程模型（Reactor 线程模型）
> https://blog.csdn.net/zhengzhaoyang122/article/details/111184617

当说到 Netty 线程模型的时候，一般首先会想到经典的 Reactor 线程模型，
尽管不同的 NIO 框架对于 Reactor 模式的实现存在差异，但本质上还是遵循了 Reactor 的基础线程模式。

#### 一、Reactor 单线程模型
Reactor 单线程模型，是指所有的 I/O操作都是在同一个 NIO线程上面完成。
NIO线程的职责如下（连接和消息应答）：
* 作为 NIO服务端，接受客户端的 TCP连接；
* 作为 NIO客户端，向服务端发起 TCP连接；
* 读取通信对端的请求和应答消息；
* 向通信对端发送消息请求或者应答消息；

Reactor 单线程模型如下图：

消息处理流程：
1）、Reactor 对象通过 select 监控连接事件，收到事件后通过 dispatcher 进行转发。
2）、如果是连接事件，则由 acceptor接收连接，并创建 handler处理后续事件。
3）、如果不是建立连接事件，则 Reactor会调用 Handler来响应。
4）、handler 会完成 read、业务处理、send的完成业务流程。

Reactor 模型中的三种角色：
①、Reactor**：负责监听和分配事件，将 I/O事件分派给对应的Handler。新的事务包含连接建立就绪、读就绪、写就绪等。
②、Acceptor：处理客户端新连接，并分派请求到处理器链中。
③、Handler：**将自身与事件绑定，执行非阻塞读写任务，完成 channel 的读入，完成处理业务逻辑后，负责将结果写出 channel。可以使用资源池来管理。

在一些小容量的应用场景下，可以使用单线程模型。
但是这对于高负载、大并发的应用场景不合适，主要原因如下：
* 一个 NIO 线程同时处理成百上千的链路，性能上无法支撑，即便 NIO 线程的 CPU 负荷达到 100% ，也无法满足海量信息的编码、解码、读取和发送。
* 当 NIO 线程负载过重之后，处理速度将变慢，这会大量客户端连接超时，超时之后往往会进行重发，这更加重了 NIO 线程的负载，最后会导致大量消息积压和处理超时，称为系统的性能瓶颈。
* 可靠性问题：一旦 NIO线程意外跑飞，或者进入死循环，会导致整个系统通信模块不可用，不能接收和处理外部消息，造成节点故障。
  为了解决这些问题，演进出了 Reactor 多线程模型，接下来就看看 Reactor 多线程模型。

#### 二、Reactor 多线程模型
与单线程模型最大的区别就是有**一组 NIO 线程来处理 I/O 操作**，它的原理图如下：

消息处理流程：
1）、Reactor 对象通过 Selector监听客户端请求事件，收到事件后通过 Dispatcher进行分发。
2）、如果是建立连接请求，则由 acceptor通过 accept处理连接请求，然后创建一个 Handler对象处理连接完成后续的各种事件。
3）、如果不是连接请求，则 Reactor 会分发给调用连接对应的 Handler 来响应。
4）、Handler 只负责响应事件，不做具体业务处理，通过 Read读取数据后，会分发给后面 Worker 线程池进行业务处理。
5）、Worker线程池分配独立的线程完成真正的业务处理，将响应结果发送给 Handler进行处理。
6）、Handler 收到响应结果后，通过 send将响应结果返回给客户端。

Reactor 多线程模型的特点如下：
* 有专门一个 NIO 线程：Acceptor 线程用于监听服务端，接收客户端的 TCP 连接请求。
* 网络IO 操作：读写等由一个 NIO 线程池负责，线程池可以采用标准的 JDK 线程池实现，它包含一个任务队列和 N个可用的线程，由这些 NIO 线程负责消息的读取、解码、编码和发送。
* 一个 NIO 线程可以同时处理 N 条链路，但是一个链路只对应一个 NIO 线程，防止发生并发操作问题。

大多数情况下，Reactor 多线程模型可以满足性能需求，但是，在个别特殊场景中，一个NIO 线程负责监听和处理所有的客户端连接可能会存在性能问题。
例如并发百万客户端连接，或者服务端需要对客户端握手进行安全认证，但是认证本身非常损耗性能。
在这类场景下，单独一个 Acceptor 线程可能会存在性能不足的问题，
为了解决性能问题，产生了第三种 Reactor 线程模型——主从 Reactor 线程模型。

### 三、主从 Reactor 多线程模型
主从 Reactor 线程模型的特点是：
服务端用于接收客户端连接的不再是一个单独的 NIO 线程，而是一个独立的 NIO 线程池。
Acceptor 接受客户端 TCP 连接请求并处理完成后（可能包含接入认证等），
将新创建的 SocketChannel 注册到 I/O 线程池（Sub reactor线程池）的某个 I/O 线程上，由它负责 SocketChannel 的读写和编解码工作。
Acceptor 线程池不仅仅用于客户端的登录、握手和安全认证，一旦链路建立成功，
就将链路注册到后端 subReactor 线程池的 I/O 线程上，由 I/O 线程负责后续的I/O 操作。

主从 Reactor 线程模型原理图：

消息处理流程：
1）、从 Acceptor Pool（主线程池 boss）中随机选择一个 Reactor 线程作为 acceptor 线程，用于绑定监听端口，接收客户端连接。
2）、acceptor 线程接收客户端连接请求之后创建新的 SocketChannel，将其注册到主线程池的其他 Reactor 线程上，由其负责接入认证、握手、黑白名单和登录等操作。
3）、第二步完成之后，业务层的链路正式建立，将 SocketChannel 从主线程池的Reactor线程的多路复用器上摘除，重新注册到工作线程池（Sub），并创建一个 Handler 用于处理各种连接事件。
4）、当有新的事件发生时，subReactor 会调用连接对应的 Handler 进行响应。
5）、Handler 通过 Read 读取数据后，会分发给后面的 Worker 线程池进行处理。
6）、Worker 线程池会分配独立的线程完成真正的业务处理，如果有响应结果则发给 Handler 进行处理。
7）、Handler 收到响应结果后通过 Send 将响应返回给 Client（结束）。

**利用主从 Reactor 线程模型，可以解决一个服务端监听线程无法有效处理所有客户端连接的性能不足问题。**
因此，在 Netty 的官方 Demo 中，推荐使用该线程模型。

### 四、Netty 的线程模型
Netty 的线程模型如下：

可以通过如下 Netty 服务端启动代码来了解它的线程模型。

服务端启动时创建两个 NioEventLoopGroup，它们实际上是两个独立的 Reactor 线程池。
一个用于接收客户端的 TCP 连接，另一个用于处理 I/O 相关的读写操作，或者执行系统 Task、定时任务 Task 等。

Netty 用于接收客户端请求的线程池职责如下：
* 接收客户端 TCP 连接，初始化 Channel 参数。
* 将链路状态变更事件通知给 ChannelPipeline。

Netty 处理 I/O 操作的 Reactor 线程池职责如下：
* 异步读取通信对端的数据报，发送读事件到 ChannelPipeline；
* 异步发送消息到通信对端，调用 ChannelPipeline 的消息发送接口；
* 执行系统调用 Task；
* 执行定时任务 Task，例如链路空闲状态监测定时任务。

通过调整线程池的线程个数，是否共享线程池等方式，
Netty 的 Reactor 线程模型可以在单线程、多线程和主从多线程间切换，
这种灵活的配置方式可以最大程度地满足不同用户的个性定制。

为了尽可能的提升性能，Netty 在很多地方进行了无锁化设计，例如在 I/O 线程内部进行串行操作，避免多线程竞争导致的性能下降问题。
表面上看，串行化设计似乎 CPU 利用率不高，并发程度不够。但是，通过调整 NIO 线程池的线程参数，
可以同时启动多个串行化的线程并行运行，这种局部无锁化的串行线程设计相比一个队列，多个工作线程的模型性能更优。
设计原理如下图所示：

Netty 的 NioEventLoop 读取到消息后，直接调用 ChannelPipeline 的 fireChannelRead（Object msg）。
只要用户不主动切换线程，一直都是 NioEventLoop 调用用户的 Handler，期间不进行线程切换。
这种串行处理方式避免了**多线程操作导致的锁的竞争**，从性能角度看是最优的。

### 五、最佳实践
Netty 的多线程编程最佳实践如下：
1）、创建两个 NioEventLoopGroup，用于逻辑隔离 NIO Acceptor 和 NIO I/O 线程。
2）、尽量不要在 ChannelHandler 中启动用户线程（解码后用于将 POJO 消息派发到后端业务线程除外）。
3）、解码要放在 NIO 线程调用的解码 Handler 中进行，不要切换到用户线程中完成消息的解码。
4）、如果业务逻辑操作非常简单、没有复杂的业务逻辑计算，没有可能会导致线程被阻塞的磁盘操作、数据库操作、网络操作等，可以直接在 NIO 线程上完成业务逻辑编排，不需要切换到用户线程。
5）、如果业务逻辑处理复杂，不要在 NIO 线程上完成，建议将编解码后的 POJO 消息封装成 Task 任务，派发到业务线程池中由业务线程执行，以保证尽快被释放，处理其他的 I/O 操作。


推荐的线程数量计算公式有以下两种：
* 线程数量 = （线程总时间/瓶颈资源时间）* 瓶颈资源的线程并行数。
* QPS(每秒查询率) = 1000/线程总时间 * 线程数。
快被释放，处理其他的 I/O 操作。
由于用户场景的不同，对于一些负责的系统，实际上很难计算出最优线程配置，只能是根据测试数据和用户场景，结合公式给出一个相对合理的范围，然后对范围内的数据进行性能测试，选择相对最优值。


