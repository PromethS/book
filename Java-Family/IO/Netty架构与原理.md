# 1. Netty概述

既然 Java 提供了 NIO，为什么还要制造一个 Netty，主要原因是 Java NIO 有以下几个缺点：

1）Java NIO 的类库和 API 庞大繁杂，使用起来很麻烦，开发工作量大。

2）使用 Java NIO，程序员需要具备高超的 Java 多线程编码技能，以及非常熟悉网络编程，比如要处理断连重连、网络闪断、半包读写、失败缓存、网络拥塞和异常流处理等一系列棘手的工作。

3）Java NIO 存在 Bug，例如 Epoll Bug 会导致 Selector 空轮训，极大耗费 CPU 资源。

Netty 对于 JDK 自带的 NIO 的 API 进行了封装，解决了上述问题，提高了 IO 程序的开发效率和可靠性，同时 Netty：

1）设计优雅，提供阻塞和非阻塞的 Socket；提供灵活可拓展的事件模型；提供高度可定制的线程模型。

2）具备更高的性能和更大的吞吐量，使用零拷贝技术最小化不必要的内存复制，减少资源的消耗。

3）提供安全传输特性。

4）支持多种主流协议；预置多种编解码功能，支持用户开发私有协议。

> **注：所谓支持 TCP、UDP、HTTP、WebSocket 等协议，就是说 Netty 提供了相关的编程类和接口**

下图为 Netty 官网给出的 Netty 架构图。

![image-20210103233244963](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210103233246.png)

我们从其中的几个关键词就能看出 Netty 的强大之处：零拷贝、可拓展事件模型；支持 TCP、UDP、HTTP、WebSocket 等协议；提供安全传输、压缩、大文件传输、编解码支持等等。

# 2. Reactor线程模型

传统的 BIO 服务端编程采用**每线程每连接**的处理模型，弊端很明显，就是面对大量的客户端并发连接时，服务端的资源压力很大；并且线程的利用率很低，如果当前线程没有数据可读，它会阻塞在 read 操作上。这个模型的基本形态如下图所示。

![企业微信截图_0a14c1a4-6602-466d-bac7-2e846eb4e7c3](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210103233428.png)

BIO 服务端编程采用的是 Reactor 模式（也叫做 Dispatcher 模式，分派模式），Reactor 模式有两个要义：

1）基于 IO **多路复用**技术，多个连接共用一个多路复用器，应用程序的线程无需阻塞等待所有连接，只需阻塞等待多路复用器即可。当某个连接上有新数据可以处理时，应用程序的线程从阻塞状态返回，开始处理这个连接上的业务。

2）基于**线程池**技术复用线程资源，不必为每个连接创建专用的线程，应用程序将连接上的业务处理任务分配给线程池中的线程进行处理，一个线程可以处理多个连接的业务。

下图反应了 Reactor 模式的基本形态：

![企业微信截图_c9225323-35ac-47db-a3ea-c2a35735ade1](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210103233529.png)

Reactor 模式有两个核心组成部分：

1）**Reactor**（图中的 ServiceHandler）：Reactor 在一个单独的线程中运行，负责监听和分发事件，分发给适当的处理线程来对 IO 事件做出反应。

2）**Handlers**（图中的 EventHandler）：处理线程执行处理方法来响应 I/O 事件，处理线程执行的是非阻塞操作。

Reactor 模式就是实现网络 IO 程序高并发特性的关键。它又可以分为单 Reactor 单线程模式、单 Reactor 多线程模式、主从 Reactor 多线程模式。

## 2.1 单 Reactor 单线程模式

单 Reactor 单线程模式的基本形态如下：

![企业微信截图_bc8758c5-8420-44c4-ad37-56995007c617](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210103233633.png)

这种模式的基本工作流程为：

1）Reactor 通过 select 监听客户端请求事件，收到事件之后通过 dispatch 进行分发

2）如果事件是建立连接的请求事件，则由 Acceptor 通过 accept 处理连接请求，然后创建一个 Handler 对象处理连接建立后的后续业务处理。

3）如果事件不是建立连接的请求事件，则由 Reactor 对象分发给连接对应的 Handler 处理。

4）Handler 会完成 read-->业务处理-->send 的完整处理流程。

这种模式的优点是：模型简单，没有多线程、进程通信、竞争的问题，一个线程完成所有的事件响应和业务处理。当然缺点也很明显：

1）存在性能问题，只有一个线程，无法完全发挥多核 CPU 的性能。Handler 在处理某个连接上的业务时，整个进程无法处理其他连接事件，很容易导致性能瓶颈。

2）存在可靠性问题，若线程意外终止，或者进入死循环，会导致整个系统通信模块不可用，不能接收和处理外部消息，造成节点故障。

单 Reactor 单线程模式使用场景为：客户端的数量有限，业务处理非常快速，比如 Redis 在业务处理的时间复杂度为 O(1)的情况。

## 2.2 单 Reactor 多线程模式

单 Reactor 单线程模式的基本形态如下：

![企业微信截图_e170f9ec-9742-47cb-9243-b7839e307af3](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210103233752.png)

这种模式的基本工作流程为：

1）Reactor 对象通过 select 监听客户端请求事件，收到事件后通过 dispatch 进行分发。

2）如果事件是建立连接的请求事件，则由 Acceptor 通过 accept 处理连接请求，然后创建一个 Handler 对象处理连接建立后的后续业务处理。

3）如果事件不是建立连接的请求事件，则由 Reactor 对象分发给连接对应的 Handler 处理。Handler 只负责响应事件，不做具体的业务处理，Handler 通过 read 读取到请求数据后，会分发给后面的 Worker **线程池**来处理业务请求。

4）Worker 线程池会分配独立线程来完成真正的业务处理，并将处理结果返回给 Handler。Handler 通过 send 向客户端发送响应数据。

这种模式的优点是可以充分的利用多核 cpu 的处理能力，缺点是多线程数据共享和控制比较复杂，Reactor 处理所有的事件的监听和响应，在单线程中运行，面对高并发场景还是容易出现性能瓶颈。

## 2.3 主从 Reactor 多线程模式

主从 Reactor 多线程模式的基本形态如下：

![企业微信截图_ec09a26a-221a-47bd-b1ff-d777a1c23ccf](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210103233916.png)

针对单 Reactor 多线程模型中，Reactor 在单个线程中运行，面对高并发的场景易成为性能瓶颈的缺陷，主从 Reactor 多线程模式让 Reactor 在多个线程中运行（分成 MainReactor 线程与 SubReactor 线程）。这种模式的基本工作流程为：

1）Reactor 主线程 **MainReactor** 对象通过 select 监听客户端连接事件，收到事件后，通过 Acceptor 处理客户端连接事件。

2）当 Acceptor 处理完客户端连接事件之后（与客户端建立好 Socket 连接），MainReactor 将连接分配给 SubReactor。（即：MainReactor 只负责监听客户端连接请求，和客户端建立连接之后将连接交由 SubReactor 监听后面的 IO 事件。）

3）SubReactor 将连接加入到自己的连接队列进行监听，并创建 Handler 对各种事件进行处理。

4）当连接上有新事件发生的时候，SubReactor 就会调用对应的 Handler 处理。

5）Handler 通过 read 从连接上读取请求数据，将请求数据分发给 Worker 线程池进行业务处理。

6）Worker 线程池会分配独立线程来完成真正的业务处理，并将处理结果返回给 Handler。Handler 通过 send 向客户端发送响应数据。

7）一个 MainReactor 可以对应多个 SubReactor，即一个 MainReactor 线程可以对应多个 SubReactor 线程。

这种模式的优点是：

1）MainReactor 线程与 SubReactor 线程的数据交互简单**职责明确**，MainReactor 线程只需要接收新连接，SubReactor 线程完成后续的**业务处理**。

2）MainReactor 线程与 SubReactor 线程的数据交互简单， MainReactor 线程只需要把新连接传给 SubReactor 线程，SubReactor 线程无需返回数据。

3）多个 SubReactor 线程能够应对**更高的并发请求**。

这种模式的缺点是**编程复杂度较高**。但是由于其优点明显，在许多项目中被广泛使用，包括 Nginx、Memcached、Netty 等。

这种模式也被叫做服务器的 **1+M+N** 线程模式，即使用该模式开发的服务器包含一个（或多个，1 只是表示相对较少）连接建立线程+M 个 IO 线程+N 个业务处理线程。这是业界成熟的服务器程序设计模式。

## 2.4 Netty的模样

Netty 的设计主要基于主从 Reactor 多线程模式，并做了一定的改进。本节将使用一种渐进式的描述方式展示 Netty 的模样，即先给出 Netty 的简单版本，然后逐渐丰富其细节，直至展示出 Netty 的全貌。

简单版本的 Netty 的模样如下：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210103234218.png)070

关于这张图，作以下几点说明：

1）BossGroup 线程维护 Selector，ServerSocketChannel 注册到这个 Selector 上，只关注连接建立请求事件（相当于主 Reactor）。

2）当接收到来自客户端的连接建立请求事件的时候，通过 ServerSocketChannel.accept 方法获得对应的 SocketChannel，并封装成 NioSocketChannel 注册到 WorkerGroup 线程中的 Selector，每个 Selector 运行在一个线程中（相当于从 Reactor）。

3）当 WorkerGroup 线程中的 Selector 监听到自己感兴趣的 IO 事件后，就调用 Handler 进行处理。

我们给这简单版的 Netty 添加一些细节：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210103234309.png)

关于这张图，作以下几点说明：

1）有两组线程池：BossGroup 和 WorkerGroup，BossGroup 中的线程（可以有多个，图中只画了一个）专门负责和客户端建立连接，WorkerGroup 中的线程专门负责处理连接上的读写。

2）BossGroup 和 WorkerGroup 含有多个不断循环的执行事件处理的线程，每个线程都包含一个 Selector，用于监听注册在其上的 Channel。

3）每个 BossGroup 中的线程循环执行以下三个步骤：

3.1）轮训注册在其上的 ServerSocketChannel 的 accept 事件（OP_ACCEPT 事件）

3.2）处理 accept 事件，与客户端建立连接，生成一个 NioSocketChannel，并将其注册到 WorkerGroup 中某个线程上的 Selector 上

3.3）再去以此循环处理任务队列中的下一个事件

4）每个 WorkerGroup 中的线程循环执行以下三个步骤：

4.1）轮训注册在其上的 NioSocketChannel 的 read/write 事件（OP_READ/OP_WRITE 事件）

4.2）在对应的 NioSocketChannel 上处理 read/write 事件

4.3）再去以此循环处理任务队列中的下一个事件

我们再来看下终极版的 Netty 的模样，如下图所示：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210103234406.png)

关于这张图，作以下几点说明：

1）Netty 抽象出两组线程池：BossGroup 和 WorkerGroup，也可以叫做 BossNioEventLoopGroup 和 WorkerNioEventLoopGroup。每个线程池中都有 NioEventLoop 线程。BossGroup 中的线程专门负责和客户端建立连接，WorkerGroup 中的线程专门负责处理连接上的读写。BossGroup 和 WorkerGroup 的类型都是 **NioEventLoopGroup**。

2）NioEventLoopGroup 相当于一个事件循环组，这个组中含有多个事件循环，每个事件循环就是一个 NioEventLoop。

3）NioEventLoop 表示一个不断循环的执行事件处理的线程，每个 NioEventLoop 都包含一个 Selector，用于监听注册在其上的 Socket 网络连接（Channel）。

4）NioEventLoopGroup 可以含有多个线程，即可以含有多个 NioEventLoop。

5）每个 BossNioEventLoop 中循环执行以下三个步骤：

5.1）**select**：轮训注册在其上的 ServerSocketChannel 的 accept 事件（OP_ACCEPT 事件）

5.2）**processSelectedKeys**：处理 accept 事件，与客户端建立连接，生成一个 NioSocketChannel，并将其注册到某个 WorkerNioEventLoop 上的 Selector 上

5.3）**runAllTasks**：再去以此循环处理任务队列中的其他任务

6）每个 WorkerNioEventLoop 中循环执行以下三个步骤：

6.1）**select**：轮训注册在其上的 NioSocketChannel 的 read/write 事件（OP_READ/OP_WRITE 事件）

6.2）**processSelectedKeys**：在对应的 NioSocketChannel 上处理 read/write 事件

6.3）**runAllTasks**：再去以此循环处理任务队列中的其他任务

7）在以上两个**processSelectedKeys**步骤中，会使用 **Pipeline**（管道），Pipeline 中引用了 Channel，即通过 Pipeline 可以获取到对应的 Channel，Pipeline 中维护了很多的处理器（拦截处理器、过滤处理器、自定义处理器等）。这里暂时不详细展开讲解 Pipeline。

# 3. 示例

下面我们写点代码来加深理解 Netty 的模样。下面两段代码分别是基于 Netty 的 TCP Server 和 TCP Client。

服务端代码为：

```java
/**
 * 需要的依赖：
 * <dependency>
 * <groupId>io.netty</groupId>
 * <artifactId>netty-all</artifactId>
 * <version>4.1.52.Final</version>
 * </dependency>
 */
public static void main(String[] args) throws InterruptedException {

    // 创建 BossGroup 和 WorkerGroup
    // 1. bossGroup 只处理连接请求
    // 2. 业务处理由 workerGroup 来完成
    EventLoopGroup bossGroup = new NioEventLoopGroup();
    EventLoopGroup workerGroup = new NioEventLoopGroup();

    try {
        // 创建服务器端的启动对象
        ServerBootstrap bootstrap = new ServerBootstrap();
        // 配置参数
        bootstrap
                // 设置线程组
                .group(bossGroup, workerGroup)
                // 说明服务器端通道的实现类（便于 Netty 做反射处理）
                .channel(NioServerSocketChannel.class)
                // 设置等待连接的队列的容量（当客户端连接请求速率大于 NioServerSocketChannel 接收速率的时候，会使用该队列做缓冲）
          			// option()方法用于给服务端的 ServerSocketChannel 添加配置
                .option(ChannelOption.SO_BACKLOG, 128)
                // 设置连接保活
                // childOption()方法用于给服务端 ServerSocketChannel
                // 接收到的 SocketChannel 添加配置
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                // handler()方法用于给 BossGroup 设置业务处理器
                // childHandler()方法用于给 WorkerGroup 设置业务处理器
                .childHandler(
                        // 创建一个通道初始化对象
                        new ChannelInitializer<SocketChannel>() {
                            // 向 Pipeline 添加业务处理器
                            @Override
                            protected void initChannel(SocketChannel socketChannel) 
                              throws Exception {
                                socketChannel.pipeline().addLast(
                                        new NettyServerHandler()
                                );
                                
                                // 可以继续调用 socketChannel.pipeline().addLast()
                                // 添加更多 Handler
                            }
                        }
                );

        System.out.println("server is ready...");

        // 绑定端口，启动服务器，生成一个 channelFuture 对象，
        // ChannelFuture 涉及到 Netty 的异步模型，后面展开讲
        ChannelFuture channelFuture = bootstrap.bind(8080).sync();
        // 对通道关闭进行监听
        channelFuture.channel().closeFuture().sync();
    } finally {
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }
}

/**
 * 自定义一个 Handler，需要继承 Netty 规定好的某个 HandlerAdapter（规范）
 * InboundHandler 用于处理数据流入本端（服务端）的 IO 事件
 * InboundHandler 用于处理数据流出本端（服务端）的 IO 事件
 */
static class NettyServerHandler extends ChannelInboundHandlerAdapter {
    /**
     * 当通道有数据可读时执行
     *
     * @param ctx 上下文对象，可以从中取得相关联的 Pipeline、Channel、客户端地址等
     * @param msg 客户端发送的数据
     * @throws Exception
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg)
            throws Exception {
        // 接收客户端发来的数据
        System.out.println("client address: "
                + ctx.channel().remoteAddress());

        // ByteBuf 是 Netty 提供的类，比 NIO 的 ByteBuffer 性能更高
        ByteBuf byteBuf = (ByteBuf) msg;
        System.out.println("data from client: "
                + byteBuf.toString(CharsetUtil.UTF_8));
    }

    /**
     * 数据读取完毕后执行
     *
     * @param ctx 上下文对象
     * @throws Exception
     */
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx)
            throws Exception {
        // 发送响应给客户端
        ctx.writeAndFlush(
                // Unpooled 类是 Netty 提供的专门操作缓冲区的工具
                // 类，copiedBuffer 方法返回的 ByteBuf 对象类似于
                // NIO 中的 ByteBuffer，但性能更高
                Unpooled.copiedBuffer(
                        "hello client! i have got your data.",
                        CharsetUtil.UTF_8
                )
        );
    }

    /**
     * 发生异常时执行
     *
     * @param ctx   上下文对象
     * @param cause 异常对象
     * @throws Exception
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
            throws Exception {
        // 关闭与客户端的 Socket 连接
        ctx.channel().close();
    }
}
```

客户端端代码为：

```java
/**
 * 需要的依赖：
 * <dependency>
 * <groupId>io.netty</groupId>
 * <artifactId>netty-all</artifactId>
 * <version>4.1.52.Final</version>
 * </dependency>
 */
public static void main(String[] args) throws InterruptedException {

    // 客户端只需要一个事件循环组，可以看做 BossGroup
    EventLoopGroup eventLoopGroup = new NioEventLoopGroup();

    try {
        // 创建客户端的启动对象
        Bootstrap bootstrap = new Bootstrap();
        // 配置参数
        bootstrap
                // 设置线程组
                .group(eventLoopGroup)
                // 说明客户端通道的实现类（便于 Netty 做反射处理）
                .channel(NioSocketChannel.class)
                // handler()方法用于给 BossGroup 设置业务处理器
                .handler(
                        // 创建一个通道初始化对象
                        new ChannelInitializer<SocketChannel>() {
                            // 向 Pipeline 添加业务处理器
                            @Override
                            protected void initChannel(
                                    SocketChannel socketChannel
                            ) throws Exception {
                                socketChannel.pipeline().addLast(
                                        new NettyClientHandler()
                                );
                                
                                // 可以继续调用 socketChannel.pipeline().addLast()
                                // 添加更多 Handler
                            }
                        }
                );

        System.out.println("client is ready...");

        // 启动客户端去连接服务器端，ChannelFuture 涉及到 Netty 的异步模型，后面展开讲
        ChannelFuture channelFuture = bootstrap.connect(
                "127.0.0.1",
                8080).sync();
        // 对通道关闭进行监听
        channelFuture.channel().closeFuture().sync();
    } finally {
        eventLoopGroup.shutdownGracefully();
    }
}

/**
 * 自定义一个 Handler，需要继承 Netty 规定好的某个 HandlerAdapter（规范）
 * InboundHandler 用于处理数据流入本端（客户端）的 IO 事件
 * InboundHandler 用于处理数据流出本端（客户端）的 IO 事件
 */
static class NettyClientHandler extends ChannelInboundHandlerAdapter {
    /**
     * 通道就绪时执行
     *
     * @param ctx 上下文对象
     * @throws Exception
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx)
            throws Exception {
        // 向服务器发送数据
        ctx.writeAndFlush(
                // Unpooled 类是 Netty 提供的专门操作缓冲区的工具
                // 类，copiedBuffer 方法返回的 ByteBuf 对象类似于
                // NIO 中的 ByteBuffer，但性能更高
                Unpooled.copiedBuffer(
                        "hello server!",
                        CharsetUtil.UTF_8
                )
        );
    }

    /**
     * 当通道有数据可读时执行
     *
     * @param ctx 上下文对象
     * @param msg 服务器端发送的数据
     * @throws Exception
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg)
            throws Exception {
        // 接收服务器端发来的数据

        System.out.println("server address: "
                + ctx.channel().remoteAddress());

        // ByteBuf 是 Netty 提供的类，比 NIO 的 ByteBuffer 性能更高
        ByteBuf byteBuf = (ByteBuf) msg;
        System.out.println("data from server: "
                + byteBuf.toString(CharsetUtil.UTF_8));
    }

    /**
     * 发生异常时执行
     *
     * @param ctx   上下文对象
     * @param cause 异常对象
     * @throws Exception
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
            throws Exception {
        // 关闭与服务器端的 Socket 连接
        ctx.channel().close();
    }
}
```

什么？你觉得使用 Netty 编程难度和工作量更大了？不会吧不会吧，你要知道，你通过这么两段简短的代码得到了一个基于主从 Reactor 多线程模式的服务器，一个高吞吐量和并发量的服务器，一个异步处理服务器……你还要怎样？

对上面的两段代码，作以下简单说明：

1）Bootstrap 和 ServerBootstrap 分别是客户端和服务器端的**引导类**，一个 Netty 应用程序通常由一个引导类开始，主要是用来配置整个 Netty 程序、设置业务处理类（Handler）、绑定端口、发起连接等。

2）客户端创建一个 NioSocketChannel 作为客户端通道，去连接服务器。

3）服务端首先创建一个 NioServerSocketChannel 作为服务器端通道，每当接收一个客户端连接就产生一个 **NioSocketChannel** 应对该客户端。

4）使用 Channel 构建网络 IO 程序的时候，不同的协议、不同的阻塞类型和 Netty 中不同的 Channel 对应，常用的 Channel 有：

- NioSocketChannel：非阻塞的 TCP 客户端 Channel（本案例的客户端使用的 Channel）

- NioServerSocketChannel：非阻塞的 TCP 服务器端 Channel（本案例的服务器端使用的 Channel）

- NioDatagramChannel：非阻塞的 UDP Channel

- NioSctpChannel：非阻塞的 SCTP 客户端 Channel

- NioSctpServerChannel：非阻塞的 SCTP 服务器端 Channel

  ......

启动服务端和客户端代码，调试以上的服务端代码，发现：

1）默认情况下 BossGroup 和 WorkerGroup 都包含 16 个线程（NioEventLoop），这是因为我的 PC 是 8 核的 NioEventLoop 的数量=**coreNum*2**。这 16 个线程相当于主 Reactor。

其实创建 BossGroup 和 WorkerGroup 的时候可以指定 NioEventLoop 数量，如下：

```
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup(16);
```

这样就能更好地分配线程资源。

2）每一个 NioEventLoop 包含如下的属性（比如自己的 Selector、任务队列、执行器等）：

![企业微信截图_bb19b716-1b9c-4999-844b-d4e81079b0d8](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210103235423.png)

3）将代码断在服务端的 NettyServerHandler.channelRead 上：

![企业微信截图_4c615000-74c7-4af3-9eb3-226e150fc666](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210103235512.png)

可以看到 ctx 中包含的属性如下：

![企业微信截图_de77dc9f-00d4-48fa-a6d7-15991888c180](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210103235522.png)

可以看到：

- 当前 ChannelHandlerContext ctx 是位于 ChannelHandlerContext **责任链**中的一环，可以看到其 next、prev 属性
- 当前 ChannelHandlerContext ctx 包含一个 Handler
- 当前 ChannelHandlerContext ctx 包含一个 Pipeline
- Pipeline 本质上是一个**双向循环链表**，可以看到其 tail、head 属性
- Pipeline 中包含一个 Channel，Channel 中又包含了该 Pipeline，两者互相引用

# 4. 核心组件

## 4.1 handler 组件

无论是服务端代码中自定义的 NettyServerHandler 还是客户端代码中自定义的 NettyClientHandler，都继承于 ChannelInboundHandlerAdapter，ChannelInboundHandlerAdapter 又继承于 ChannelHandlerAdapter，ChannelHandlerAdapter 又实现了` ChannelHandler`：

```java
public class ChannelInboundHandlerAdapter 
    extends ChannelHandlerAdapter 
    implements ChannelInboundHandler {
    ......
public abstract class ChannelHandlerAdapter 
    implements ChannelHandler {
    ......
```

因此无论是服务端代码中自定义的 NettyServerHandler 还是客户端代码中自定义的 NettyClientHandler，都可以统称为 ChannelHandler。

Netty 中的 ChannelHandler 的作用是，在当前 ChannelHandler 中处理 IO 事件，并将其传递给 ChannelPipeline 中下一个 ChannelHandler 处理，因此多个 ChannelHandler 形成一个责任链，责任链位于 ChannelPipeline 中。

数据在基于 Netty 的服务器或客户端中的处理流程是：读取数据-->解码数据-->处理数据-->编码数据-->发送数据。其中的每个过程都用得到 ChannelHandler 责任链。

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210110161315.png)

ChannelPipeline 提供了 ChannelHandler 链的容器。以客户端应用程序为例，如果事件的方向是从客户端到服务器的，我们称事件是出站的，那么客户端发送给服务器的数据会通过 Pipeline 中的一系列 ChannelOutboundHandler 进行处理；如果事件的方向是从服务器到客户端的，我们称事件是入站的，那么服务器发送给客户端的数据会通过 Pipeline 中的一系列 ChannelInboundHandler 进行处理。

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210110161401.png)

无论是服务端代码中自定义的 NettyServerHandler 还是客户端代码中自定义的 NettyClientHandler，都继承于 ChannelInboundHandlerAdapter，ChannelInboundHandlerAdapter 提供的方法如下：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210110161539.png)

从方法名字可以看出，它们在不同的事件发生后被触发，例如注册 Channel 时执行 channelRegistred()、添加 ChannelHandler 时执行 handlerAdded()、收到入站数据时执行 channelRead()、入站数据读取完毕后执行 channelReadComplete()等等。

## 4.2 Pipeline 组件

Netty 的 ChannelPipeline，它维护了一个 ChannelHandler 责任链，负责拦截或者处理 inbound（入站）和 outbound（出站）的事件和操作。这一节给出更深层次的描述。

ChannelPipeline 实现了一种高级形式的拦截过滤器模式，使用户可以完全控制事件的处理方式，以及 Channel 中各个 ChannelHandler 如何相互交互。

每个 Netty Channel 包含了一个 ChannelPipeline（其实 Channel 和 ChannelPipeline 互相引用），而 ChannelPipeline 又维护了一个由 ChannelHandlerContext 构成的双向循环列表，其中的每一个 ChannelHandlerContext 都包含一个 ChannelHandler。（前文描述的时候为了简便，直接说 ChannelPipeline 包含了一个 ChannelHandler 责任链，这里给出完整的细节。）

如下图所示（图片来源于网络）：

![企业微信截图_174f28ff-181d-4e7d-a98d-204dc30c8fb6](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210110161820.png)

还记得下面这张图吗？这是上文中基于 Netty 的 Server 程序的调试截图，可以从中看到 ChannelHandlerContext 中包含了哪些成分：

![企业微信截图_b1f8f517-1497-4361-839e-fe2162ef142c](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210110161902.png)

ChannelHandlerContext 除了包含 ChannelHandler 之外，还关联了对应的 Channel 和 Pipeline。可以这么来讲：ChannelHandlerContext、ChannelHandler、Channel、ChannelPipeline 这几个组件之间互相引用，互为各自的属性，你中有我、我中有你。

在处理入站事件的时候，入站事件及数据会从 Pipeline 中的双向链表的头 ChannelHandlerContext 流向尾 ChannelHandlerContext，并依次在其中每个 ChannelInboundHandler（例如解码 Handler）中得到处理；出站事件及数据会从 Pipeline 中的双向链表的尾 ChannelHandlerContext 流向头 ChannelHandlerContext，并依次在其中每个 ChannelOutboundHandler（例如编码 Handler）中得到处理。

## 4.3 EventLoopGroup 组件

在基于 Netty 的 TCP Server 代码中，包含了两个 EventLoopGroup——bossGroup 和 workerGroup，EventLoopGroup 是一组 EventLoop 的抽象。

追踪 Netty 的 EventLoop 的继承链，可以发现 EventLoop 最终继承于 JUC Executor，因此 EventLoop 本质就是一个 JUC Executor，即线程，JUC Executor 的源码为：

```java
public interface Executor {
    /**
     * Executes the given command at some time in the future.
     */
    void execute(Runnable command);
}
```

Netty 为了更好地利用多核 CPU 的性能，一般会有多个 EventLoop 同时工作，每个 EventLoop 维护着一个 Selector 实例，Selector 实例监听注册其上的 Channel 的 IO 事件。

EventLoopGroup 含有一个 next 方法，它的作用是按照一定规则从 Group 中选取一个 EventLoop 处理 IO 事件。

在服务端，通常 Boss EventLoopGroup 只包含一个 Boss EventLoop（单线程），该 EventLoop 维护者一个注册了 ServerSocketChannel 的 Selector 实例。该 EventLoop 不断轮询 Selector 得到 OP_ACCEPT 事件（客户端连接事件），然后将接收到的 SocketChannel 交给 Worker EventLoopGroup，Worker EventLoopGroup 会通过 next()方法选取一个 Worker EventLoop 并将这个 SocketChannel 注册到其中的 Selector 上，由这个 Worker EventLoop 负责该 SocketChannel 上后续的 IO 事件处理。整个过程如下图所示：

![企业微信截图_399d8775-148d-4aee-af3a-fe3abd243bf2](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210110162316.png)

## 4.4 TaskQueue

在 Netty 的每一个 NioEventLoop 中都有一个 TaskQueue，设计它的目的是在任务提交的速度大于线程的处理速度的时候起到**缓冲作用**。或者用于**异步**地处理 Selector 监听到的 IO 事件。

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210110162401.png)

Netty 中的任务队列有三种使用场景：

1）处理用户程序的自定义普通任务的时候

2）处理用户程序的自定义定时任务的时候

3）非当前 Reactor 线程调用当前 Channel 的各种方法的时候。

对于第一种场景，举个例子，2.4 节的基于 Netty 编写的服务端的 Handler 中，假如 channelRead 方法中执行的过程很耗时，那么以下的阻塞式处理方式无疑会降低当前 NioEventLoop 的并发度：

```java
/**
 * 当通道有数据可读时执行
 *
 * @param ctx 上下文对象
 * @param msg 客户端发送的数据
 * @throws Exception
 */
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg)
        throws Exception {
    // 借助休眠模拟耗时操作
    Thread.sleep(LONG_TIME);

    ByteBuf byteBuf = (ByteBuf) msg;
    System.out.println("data from client: "
            + byteBuf.toString(CharsetUtil.UTF_8));
}
```

改进方法就是借助任务队列，代码如下：

```java
/**
 * 当通道有数据可读时执行
 *
 * @param ctx 上下文对象
 * @param msg 客户端发送的数据
 * @throws Exception
 */
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg)
        throws Exception {
    // 假如这里的处理非常耗时，那么就需要借助任务队列异步执行

    final Object finalMsg = msg;

    // 通过 ctx.channel().eventLoop().execute()将耗时
    // 操作放入任务队列异步执行
    ctx.channel().eventLoop().execute(new Runnable() {
        public void run() {
            // 借助休眠模拟耗时操作
            try {
                Thread.sleep(LONG_TIME);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            ByteBuf byteBuf = (ByteBuf) finalMsg;
            System.out.println("data from client: "
                    + byteBuf.toString(CharsetUtil.UTF_8));
        }
    });
    
    // 可以继续调用 ctx.channel().eventLoop().execute()
    // 将更多操作放入队列
    
    System.out.println("return right now.");
}
```

断点跟踪这个函数的执行，可以发现该耗时任务确实被放入的当前 NioEventLoop 的 taskQueue 中了。

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210110162707.png)

对于第二种场景，举个例子，2.4 节的基于 Netty 编写的服务端的 Handler 中，假如 channelRead 方法中执行的过程并不需要立即执行，而是要定时执行，那么代码可以这样写：

```java
/**
 * 当通道有数据可读时执行
 *
 * @param ctx 上下文对象
 * @param msg 客户端发送的数据
 * @throws Exception
 */
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg)
        throws Exception {

    final Object finalMsg = msg;

    // 通过 ctx.channel().eventLoop().schedule()将操作
    // 放入任务队列定时执行（5min 之后才进行处理）
    ctx.channel().eventLoop().schedule(new Runnable() {
        public void run() {

            ByteBuf byteBuf = (ByteBuf) finalMsg;
            System.out.println("data from client: "
                    + byteBuf.toString(CharsetUtil.UTF_8));
        }
    }, 5, TimeUnit.MINUTES);
    
    // 可以继续调用 ctx.channel().eventLoop().schedule()
    // 将更多操作放入队列

    System.out.println("return right now.");
}
```

断点跟踪这个函数的执行，可以发现该定时任务确实被放入的当前 NioEventLoop 的 scheduleTaskQueue 中了。

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210110162752.png)

对于第三种场景，举个例子，比如在基于 Netty 构建的推送系统的业务线程中，要根据用户标识，找到对应的 SocketChannel 引用，然后调用 write 方法向该用户推送消息，这时候就会将这一 write 任务放在任务队列中，write 任务最终被异步消费。这种情形是对前两种情形的应用，且涉及的业务内容太多，不再给出示例代码，读者有兴趣可以自行完成，这里给出以下提示：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210110162927.png)

## 4.5 Future 和 Promise

Netty**对使用者提供的多数 IO 接口（即 Netty Channel 中的 IO 方法）**是异步的（**即都立即返回一个 Netty Future，而 IO 过程异步进行**），因此，调用者调用 IO 操作后是不能直接拿到调用结果的。要想得到 IO 操作结果，可以借助 Netty 的 Future（上面代码中的 ChannelFuture 就继承了 Netty Future，Netty Future 又继承了 JUC Future）查询执行状态、等待执行结果、获取执行结果等，使用过 JUC Future 接口的同学会非常熟悉这个机制，这里不再展开描述了。也可以通过 Netty Future 的 addListener()添加一个回调方法来异步处理 IO 结果，如下：

```java
// 启动客户端去连接服务器端
// 由于 bootstrap.connect()是一个异步操作，因此用.sync()等待
// 这个异步操作完成
final ChannelFuture channelFuture = bootstrap.connect(
        "127.0.0.1",
        8080).sync();

channelFuture.addListener(new ChannelFutureListener() {
    /**
     * 回调方法，上面的 bootstrap.connect()操作执行完之后触发
     */
    public void operationComplete(ChannelFuture future)
            throws Exception {
        if (channelFuture.isSuccess()) {
            System.out.println("client has connected to server!");
            // TODO 其他处理
        } else {
            System.out.println("connect to serverfail!");
            // TODO 其他处理
        }
    }
});
```

Netty Future 提供的接口有：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210110163145.png)

> 注：会有一些资料给出这样的描述：“Netty 中所有的 IO 操作都是异步的”，这显然是错误的。Netty 基于 Java NIO，Java NIO 是同步非阻塞 IO。Netty 基于 Java NIO 做了封装，向使用者提供了异步特性的接口，因此本文说 Netty**对使用者提供的多数 IO 接口（即 Netty Channel 中的 IO 方法）**是异步的。例如在 io.netty.channel.ChannelOutboundInvoker（Netty Channel 的 IO 方法多继承于此）提供的多数 IO 接口都返回 Netty Future：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210110163209.png)

Promise 是可写的 Future，Future 自身并没有写操作相关的接口，Netty 通过 Promise 对 Future 进行扩展，用于设置 IO 操作的结果。Future 继承了 Future，相关的接口定义如下图所示，相比于上图 Future 的接口，它多出了一些 setXXX 方法：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210110163846.png)

Netty 发起 IO 写操作的时候，会创建一个新的 Promise 对象，例如调用 ChannelHandlerContext 的 write(Object object)方法时，会创建一个新的 ChannelPromise，相关代码如下：

```java
@Override
public ChannelFuture write(Object msg) {
    return write(msg, newPromise());
}
......
@Override
public ChannelPromise newPromise() {
    return new DefaultChannelPromise(channel(), executor());
}
......
```

当 IO 操作发生异常或者完成时，通过 Promise.setSuccess()或者 Promise.setFailure()设置结果，并通知所有 Listener。

# 5. 源码解析

详见：https://mp.weixin.qq.com/s/F3uUqgEMxX3-sQPFHVFGow

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210110220054.png)

# 6. 常见问题

## 6.1 拆包和粘包

TCP是个“流”协议，所谓流，就是没有界限的一串数据。TCP底层并不了解上层业务数据的具体含义，它会根据TCP缓冲区的实际情况进行包的划分，所以在业务上认为，一个完整的包可能会被TCP拆分成多个包进行发送，也有可能把多个小的包封装成一个大的数据包发送，这就是所谓的TCP粘包和拆包问题。

数据从发送方到接收方需要经过操作系统的缓冲区，而造成粘包和拆包的主要原因就在这个缓冲区上。粘包可以理解为缓冲区数据堆积，导致多个请求数据粘在一起，而拆包可以理解为发送的数据大于缓冲区，进行拆分处理。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210116180243.PNG)

详细来说，造成粘包和拆包的原因主要有以下三个：

1. 应用程序write写入的字节大小大于套接口发送缓冲区大小
2. 进行MSS大小的TCP分段
3. 以太网帧的payload大于MTU进行IP分片。

**粘包和拆包的解决方法：**

由于底层的TCP无法理解上层的业务数据，所以在底层是无法保证数据包不被拆分和重组的，这个问题只能通过上层的应用协议栈设计来解决，根据业界的主流协议的解决方案，可以归纳如下。

1. **消息长度固定**，累计读取到长度和为定长LEN的报文后，就认为读取到了一个完整的信息
2. 将**回车换行符**作为消息结束符
3. 将**特殊的分隔符**作为消息的结束标志，回车换行符就是一种特殊的结束分隔符
4. 通过在消息头中**定义长度字段**来标识消息的总长度

**Netty中的粘包和拆包解决方案：**

针对上一小节描述的粘包和拆包的解决方案，对于拆包问题比较简单，用户可以自己定义自己的编码器进行处理，Netty并没有提供相应的组件。对于粘包的问题，由于拆包比较复杂，代码比较处理比较繁琐，Netty提供了4种解码器来解决，分别如下：

1. 固定长度的拆包器 FixedLengthFrameDecoder，每个应用层数据包的都拆分成都是固定长度的大小
2. 行拆包器 LineBasedFrameDecoder，每个应用层数据包，都以换行符作为分隔符，进行分割拆分
3. 分隔符拆包器 DelimiterBasedFrameDecoder，每个应用层数据包，都通过自定义的分隔符，进行分割拆分
4. 基于数据包长度的拆包器 LengthFieldBasedFrameDecoder，将应用层数据包的长度，作为接收端应用层数据包的拆分依据。按照应用层数据包的大小，拆包。这个拆包器，有一个要求，就是应用层协议中包含数据包的长度