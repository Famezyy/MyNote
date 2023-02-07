# 第03章_Netty核心原理与基础实战

Netty 是一个 Java NIO 客户端/服务器框架，是一个为了快速开发可维护的高性能、高可扩展的网络服务器和客户端程序而提供的异步事件驱动基础框架和工具。

Netty 极大地简化了网络编程流程，例如 TCP、UDP 套接字和 HTTP Web 服务程序的开发。还可以轻松地开发应用层协议的通信程序，如 FTP、SMTP、HTTP 以及其他的传统应用层协议。同时提供了高性能和高扩展性，它基于 Java 的 NIO 设计了一套优秀的、高性能的 Reactor 模式实现。

## 1.入门案例：DiscardServer

功能：读取客户端的输入数据，不给客户端任何回复。

首先通过 maven 导入 Netty 的依赖坐标，建议使用 4.0 以上的版本。

```xml
<!-- https://mvnrepository.com/artifact/io.netty/netty-all -->
<dependency>
  <groupId>io.netty</groupId>
  <artifactId>netty-all</artifactId>
  <version>4.1.87.Final</version>
</dependency>
```

创建一个服务端类 NettyDiscardServer

```java
public class NettyDiscardServer {

  Logger logger = Logger.getLogger("nettyDiscardServer");
  private final int serverPort;
  ServerBootstrap b = new ServerBootstrap();

  public NettyDiscardServer(int port) {
    this.serverPort = port;
  }

  public void runServer() {
    // 创建reactor 线程组
    EventLoopGroup bossLoopGroup = new NioEventLoopGroup(1);
    EventLoopGroup workerLoopGroup = new NioEventLoopGroup();

    try {
      // 1 设置 reactor 线程组
      b.group(bossLoopGroup, workerLoopGroup);
      // 2 设置 nio 类型的 channel
      b.channel(NioServerSocketChannel.class);
      // 3 设置监听端口
      b.localAddress(serverPort);
      // 4 设置通道的参数
      b.option(ChannelOption.SO_KEEPALIVE, true);
      b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
      b.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
      b.childOption(ChannelOption.TCP_NODELAY, true);

      // 5 装配子通道流水线
      b.childHandler(new ChannelInitializer<SocketChannel>() {
        // 有连接到达时会创建一个 channel
        @Override
        protected void initChannel(SocketChannel ch) throws Exception {
          // pipeline 管理子通道 channel 中的 Handler
          // 向子 channel 流水线添加一个 handler 处理器
          ch.pipeline().addLast(new NettyDiscardHandler());
        }
      });
      // 6 开始绑定 server
      ChannelFuture channelFuture = b.bind();

      channelFuture.addListener(l -> {

        if (l.isSuccess())
          logger.info(" 服务器端口绑定 正常: " + channelFuture.channel().localAddress());
        else
          logger.info(" 服务器端口绑定 失败: " + channelFuture.channel().localAddress());

      });
      // 通过调用 sync 同步方法阻塞直到绑定成功
      channelFuture.sync();

      logger.info(" 服务器启动成功，监听端口: " + channelFuture.channel().localAddress());

      // 7 等待通道关闭的异步任务结束
      // 服务监听通道会一直等待通道关闭的异步任务结束
      ChannelFuture closeFuture = channelFuture.channel().closeFuture();
      closeFuture.sync();
    } catch (Exception e) {
      e.printStackTrace();
    } finally {
      // 8 优雅关闭 EventLoopGroup，
      // 释放掉所有资源包括创建的线程
      workerLoopGroup.shutdownGracefully();
      bossLoopGroup.shutdownGracefully();
    }

  }

  public static void main(String[] args) throws InterruptedException {
    int port = 18888;
    new NettyDiscardServer(port).runServer();
  }

}
```

首先要说的是 Reactor 模式中的 Reactor 组件。Netty 中对应的反应器组件有多种，一般来说对于多线程的 Java NIO 通信的应用场景，Netty 对应的反应器组件为`NioEventLoopGroup`。

在上面的例子中，使用了两个`NioEventLoopGroup`反应器组件实例：第一个负责服务器通道新连接的 IO 事件的监听，第二个主要负责传输通道的 IO 事件的处理和数据传输。

其次要说的是 Reactor 模式中的 Handler 组件。所有业务处理都在 handler 中完成。

再次，还用到了 Netty 的服务引导类`ServerBootstrap`。服务引导类是一个组装和集成器，作用是将不同的 Netty 组件组装在一起。`ServerBootstrap`能够按照应用场景的需要为组件设置好基础性的参数。

创建一个 NettyDiscardHandler

```java
class NettyDiscardHandler extends ChannelInboundHandlerAdapter {

  Logger logger = Logger.getLogger("nettyDiscardHandler");

  @Override
  public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

    ByteBuf in = (ByteBuf) msg;
    try {
      logger.info("收到消息,丢弃如下:");
      while (in.isReadable()) {
        System.out.print((char) in.readByte());
      }
      System.out.println();

    } finally {
      ReferenceCountUtil.release(msg);
    }
  }
}
```

Netty 的 Handler 需要处理多种 IO 事件（如读就绪、写就绪），对应于不同的 IO 事件，Netty 提供了一些基础方法。这些方法都已经提前封装好，应用程序直接继承或者实现即可。例如，对应于处理入站的 IO 事件，对应的接口为`ChannelInboundHandler`，并且 Netty 提供了`ChannelInboundHandlerAdapter`适配器作为入站处理器的默认实现。

>   **说明：关于入站和出站**
>
>   入站指的是输入，出站指的是输出。Netty 的出/入站与 Java NIO 中的略有不同，Netty 的出站可以理解为从 handler 传递到 Channel 的操作，比如 write 写通道，read 读通道数据。Netty 的入站可以理解为从 Channel 传递到 Handler 的操作，比如 Channel 数据过来之后会触发 Handler 的`channelRead()`入站处理方法。

如果要实现自己的入站处理器，可以简单地继承`ChannelInboundHandlerAdapter`入站处理器适配器，重写通道读取方法`channelRead()`。

## 2.Netty的Reactor模式

### 2.1 Reactor模式中IO事件处理流程

Channel - Selector - Reactor - Handler

Reactor 模式中 IO 事件的处理流程大致分为 4 步：

-   **注册通道**：IO 事件源于通道，一个 IO 事件一定属于某个通道，如果要查询通道的事件，首先就要讲通道注册到选择器
-   **查询事件**：一个线程负责一个反应器，通过不断的轮询来查询选择器中的 IO 事件（选择键）
-   **分发事件**：如果查询到 IO 事件，则分发给与 IO 事件有绑定关系的 Handler 业务处理器
-   完成真正的 IO 操作和业务处理，由 Handler 业务处理器负责

### 2.2 Netty中的Channel

Netty 对 Java NIO 的 Channel 组件进行了封装，对每一种通信连接协议，Netty 都实现了自己的通道，每一种协议基本上都有 NIO 和 OIO 两个版本。

Netty 常见的通道如下：

-   `NioSocketChannel`：异步非阻塞 TCP Socket 传输通道
-   `NioServerSocketChanel`：异步非阻塞 TCP Socket 服务端监听通道
-   `NioDatagramChannel`：异步非阻塞的 UDP 传输通道
-   `NioSctpChannel`：异步非阻塞 Sctp 传输通道
-   `NioSctpServerChannel`：异步非阻塞 Sctp 服务端监听通道
-   `OioSocketChannel`：同步阻塞式 TCP Socket 传输通道
-   `OioServerSocketChannel`：同步阻塞式 TCP Socket 服务端监听通道
-   `OioDatagramChannel`：同步阻塞式 UDP 传输通道
-   `OioSctpChannel`：同步阻塞式 Sctp 传输通道
-   `OioSctpServerChannel`：同步阻塞式 Sctp 服务端监听通道

一般服务端编程用到最多的通信协议还是 TCP，对应的 Netty 传输通道类型为`NioSocketChannel`，Netty 服务器监听通道类型为`NioServerSocketChannel`。不论哪种通道，在主要的 API 和使用方式上基本相同，更多的只是底层的传输协议不同。

在 Netty 的`NioSocketChannel`内部封装了一个 Java NIO 的`SelectableChannel`成员，对 Netty 的`NioSocketChannel`通道上的所有 IO 操作最终都会落地到 Java NIO 的`SelectableChannel`底层通道。`NioSocketChannel`的继承关系图如下：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302042021560.png" alt="image-20230204202121542" style="zoom: 33%;" />

### 2.3 Netty中的Reactor

在 Reactor 模式中，一个反应器（或者 SubReactor 子反应器）会由一个事件处理线程负责事件查询和分发。该线程不断进行轮询，通过 Selector 选择器不断查询注册过的 IO 事件。如果查询到 IO 事件，就分发给 Handler 业务处理器。

Netty 中的反应器组件有多个实现类，这些实现类与其通道类型相互匹配。例如，`NioSocketChannel`通道的相应的反应器类为`NioEventLoop`（Nio 事件轮询）。

`NioEventLoop`类有两个重要的成员属性：一个是 Thread 线程类的成员，一个是 Java NIO 选择器的成员属性。`NioEventLoop`的继承关系如下：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302042022822.png" alt="image-20230204202208806" style="zoom: 33%;" />

可以看出，一个`NioEventLoop`拥有一个线程，负责一个 Java NIO 选择器的 IO 事件轮询。而一个 EventLoop 反应器可以注册成千上万的通道。

### 2.4 Netty中的Handler

Java NIO 中可供选择器监控的通道 IO 事件类型有以下 4 种：

-   `SelectionKey.OP_READ`：可读
-   `SelectionKey.OP_WRITE`：可写
-   `SelectionKey.OP_CONNECT`：连接
-   `SelectionKey.OP_ACCEPT`：接收

在 Netty 中，EventLoop 反应器内部有一个线程负责 Java NIO 选择器的事件的轮询，然后进行对应的事件分发。事件分发的目标就是 Netty 的 Handler。

Netty 的 Handler 分为两大类：第一类是`ChannelInboundHandler`入站处理器；第二类是`ChannelOutboundHandler`出站处理器，二则都继承了 ChannelHandler 处理器接口。

**入站处理**

当通道中发生了 OP_READ 事件后，会被 EventLoop 查询到，然后分发给`ChannelInboundHandler`入站处理器，调用对应的入站处理的`read()`方法，从通道中读取数据。Netty 中的入站处理不仅仅是 OP_READ 输入事件的处理，还包括从通道底层触发，由 Netty 通过层层传递，调用`ChannelInboundHandler`入站处理器进行的其他某个处理。

**出站处理**

Netty 中的出站处理指的是从`ChannelOutboundHandler`出站处理器到通道的某次 IO 操作。例如，在应用程序完成业务处理后，可以通过`ChannelOutboundHandler`出站处理器将处理的结果写入底层通道。最常用的一个方法就是`write()`，即把数据写入通道。

Netty 中的出站处理不仅仅包括 Java NIO 的 OP_WRITE 可写事件，还包括 Netty 自身从处理器到通道方向的其他操作。

无论入站还是出站，Netty 都提供了各自的默认适配器实现：`ChannelInboundHandler`的默认实现为`ChannelInboundHandlerAdapter`；`ChannelOutboundHandler`的默认实现为`ChannelOutboundHandlerAdapter`。如果要实现自己的业务处理器，只需要继承通道处理适配器即可。

### 2.5 Netty中的Pipeline

先梳理一下 Netty 的 Reactor 模式实现中各个组件之间的关系：

-   反应器和通道之间是一对多的关系：一个反应器可以查询很多个通道的 IO 事件
-   通道和反应器之间是一对一的关系：一个通道之对应一个反应器
-   通道和 Handler 处理器实例之间是多对多的关系：一个通道的 IO 事件可以被多个 Handler 实例处理；一个 Handler 处理器实例也能绑定到很多通道，处理多个通道的 IO 事件

为了处理通道和 Handler 处理器实例之间的绑定关系，Netty 设计了`ChannelPipeline`（通道流水线）。它像一条通道，将绑定到一个通道的多个 Handler 处理器实例串联在一起，形成一条流水线。`ChannelPipeline`的默认实现实际上被设计成一个双向链表。所有的 Handler 处理器实例被包装成双向链表的节点，被加入到`ChannelPipeline`中。

>   **说明**
>
>   一个 Netty 通道拥有一个`ChannelPipeline`类型的成员属性，该属性名叫做 pipeline。

在向后流动的过程中，会出现 3 种情况：

-   如果后面还有其他 Handler 入站处理器，那么 IO 事件可以交给下一个 Handler 处理器向后流动
-   如果后面没有其他的入站处理器，就意味着这个 IO 事件在此次流水线的中的处理结束了
-   如果在中间需要终止流动，可以选择不将 IO 事件交给下一个 Handler 处理器，流水线的执行也会终止

Netty 的通道流水线是双向的，入站处理器的执行次序是从前到后，出站处理器的执行次序是从后到前。IO 事件在流水线上的执行次序与 IO 事件的类型是有关系的。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302042100927.png" alt="image-20230204210038906" style="zoom:50%;" />

除了流动的方向与 IO 操作类型有关之外，流动过程中所经过的处理器类型也是与 IO 操作的类型有关。入站的 IO 操作只能从 Inbound 入站处理器类型的 Handler 流过；出站的 IO 操作只能从 Outbound 出站处理器类型的 Handler 流过。

为了方便开发，Netty 提供了一系类辅助引导类，服务端的引导类叫做`ServerBootstrap`，客户端的引导类叫做`Bootstrap`。

## 3.详解Bootstrap

Bootstrap 类是 Netty 提供的一个便利的工厂类，可以通过它来完成 Netty 的客户端或服务端的 Netty 组件的组装，以及 Netty 程序的初始化和启动执行。如果不用 Bootstrap 类，则需要手动创建通道、完成各种设置和启动注册到 EventLoop 反应器，然后开始事件的轮询和处理，这个过程会非常麻烦。

在 Netty 中有两个引导类，`Bootstrap`是 client 专用，`ServerBootstrap`是 server 专用。他们大致的配置和使用方法都是相同的。下面主要介绍`ServerBootstrap`。

在介绍`ServerBootstrap`服务器启动流程之前，首先介绍两个基础概念：**父子通道**和**EventLoopGroup**（事件轮询线程组）。

### 3.1 父子通道

在 Netty 中，每一个`NioSocketChannel`通道所封装的都是 Java NIO 通道，再往下就对应到了操作系统底层的 socket 文件描述符。理论上，操作系统底层的 socket 文件描述符分为两类：

-   **连接监听类型**

    连接监听类型的 socket 描述符处于服务端，负责接收客户端的套接字连接；在服务端，一个“连接监听类型”的 socket 描述符可以接受成千上万的传输类的 socket 文件描述符。

-   **数据传输类型**

    数据传输类的 socket 描述符负责传输数据。同一条 TCP 的 socket 传输链路在服务器和客户端都分别会有一个与之相对应的数据传输类型的 socket 文件描述符。

在 Netty 中，异步非阻塞的服务端监听通道`NioServerSocketChannel`所封装的 Linux 底层的文件描述符是“连接监听类型”的 socket 描述符；异步非阻塞的传输通道`NioSocketChannel`所封装的 Linux 的文件描述符是“数据传输类型”的 socket 描述符。

在 Netty 中，将有接收关系的监听通道和传输通道叫做父子通道。其中，负责服务器连接监听和接收的监听通道（如`NioServerSocketChannel`）也叫父通道，对应于每一个接收到的传输类通道（如`NioSocketChannel`）也叫子通道。

### 3.2 EventLoopGroup

Netty 中的 Reactor 模式实现是多线程版本的，一个 EventLoop 相当于一个子反应器，一个子反应器拥有一个事件轮询线程，同时拥有一个 Java NIO 选择器。而 Netty 中使用了`EventLoopGroup`（事件轮询组），将多个 EventLoop 线程放在一起，实现了多线程版本的 Reactor 模式。

`EventLoopGroup`的构造函数有一个参数，用于指定内部的线程数。在构造器初始化时，会按照传入的线程数量在内部构造多个线程，对应多个 EventLoop 子反应器，进行多线程的 IO 事件查询和分发。如果没有传入参数或传入 0，默认线程数量为最大可用的 CPU 处理器核心数量的 2 倍。

在之前的 Reactor 实战中，服务端一般有两个独立的反应器，一个负责新连接的监听和接收，另一个负责 IO 事件轮询和分发，并且两个反应器相互隔离。对应到 Netty 服务器程序，则需要设置两个`EventLoopGroup`，具体职责如下：

-   负责新连接的监听和接收的`EventLoopGroup`中的反应器完成查询通道的新连接 IO 事件查询，戏称为“Boss”轮询组
-   负责 IO 事件轮询和分发的反应器完成查询所有子通道的 IO 事件，并且执行对应的 Handler 处理器完成 IO 处理——例如数据的输入和输出，戏称为“Worker“轮询组

Netty 的`EventLoopGroup`与`EventLoop`、`Channel`之间的关系如图：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302042142153.png" alt="image-20230204214244133" style="zoom:50%;" />

### 3.3 Bootstrap启动流程

Bootstrap 的启动流程也就是 Netty 组件的组装、配置，以及 Netty 服务器活着客户端的启动流程，大致分为 8 个步骤。接下来重点演示服务端引导类`ServerBootstrap`的使用。

```java
// 创建一个服务端的引导类
ServerBootstrap b = new ServerBootstrap();
```

1.   创建反应器轮询组，并设置到`ServerBootstrap`引导类实例

     ```java
     // 创建反应器轮询组
     // boss 轮询组，负责监听连接 IO 事件
     EventLoopGroup bossLoopGroup = new NioEventLoopGroup(1);
     
     // worker 轮询组，负责数据传输事件和处理
     EventLoopGroup workerLoopGroup = new NioEventLoopGroup();
     
     // 将 EventLoopGroup 添加进引导类
     b.group(bossLoopGroup, workerLoopGroup);
     ```

     可以不需要分开监听新连接事件和输出事件，可以仅配置一个`EventLoopGroup`反应器轮询组，即调用`b.group(workerGroup)`。在这种模式下，新连接监听 IO 事件和数据传输 IO 事件可能被挤在同一个线程中处理。这样的话新连接的接收可能会被更加耗时的数据传输或者业务处理所阻塞。所以一般建议设置成两个轮询组的工作模式。

2.   设置通道的 IO 类型

     ```java
     // 设置传输通道类型为 NIO
     b.channel(NioServerSocketChannel.class);
     ```

     可传入`OioServerSocketChannel.class`来制定 BIO 类型（不建议）。

3.   设置服务器监听端口

     ```java
     b.localAddress(new InetSocketAddress(port))；
     ```

4.   设置传输通道的配置选项

     ```java
     // 开启 TCP 底层心跳机制
     b.option(ChannelOption.SO_KEEPALIVE, true);
     b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
     ```

     对于服务器的 Bootstrap 而言，这个方法会给父通道设置一些与传输协议相关的选项。如果要给子通道设置一些通道选项，则需要调用`childOption()`方法。

5.   装配子通道的 Pipeline

     每一个通道都用一条`ChannelPipeline`流水线，它的内部有一个双向的链表。装配流水线的方式是：将业务处理器`ChannelHandler`实例包装后加入双向链表。

     装配子通道的 Handler 流水线需要调用引导类的`childHandler()`方法，该方法传入一个`ChannelInitializer`通道初始化类的实例作为参数。每当父通道成功接收到一个连接并创建成功一个子通道后，就会初始化子通道，此时就会调用`ChannelInitializer`实例。

     在`ChannelInitializer`通道初始化类的实例中，有一个`initChannel`初始化方法，在子通道创建后会被执行，可以向子通道流水线增加业务处理器。

     ```java
     b.childHandler(new Channelinitializer<SocketChannel>() {
       // 有连接到达时会创建一个通道的子通道，并初始化
       protected void initChannel(SocketChannel ch) {
         // 这里可以管理子通道中的 Handler 业务处理器
         // 向子通道流水线添加一个 Handler 业务处理器
         ch.pipeline().addList(new NettyDiscardHandler());
       }
     });
     ```

     >   **注意**
     >
     >   `Channelinitializer`处理器有一个范型参数`SocketChannel`，它代表需要初始化的通道类型，这个类型需要和前面的引导类中设置的子通道类型对应起来。

     >   **为什么只装配子通道的流水线？**
     >
     >   父通道`NioServerSocketChannel`的内部业务处理是固定的：接受新连接后，创建子通道，然后初始化子通道，所以不需要特别的配置，由 Netty 自行进行装配。如果需要完成特殊的父通道业务处理，可以类似地调用`ServerBootstrap`的`handler(ChannelHandler handler)`方法，为父通道设置`ChannelInitializer`初始化器。

6.   开始绑定服务器新连接的监听端口

     ```java
     // 绑定端口，调用 sync() 同步方法阻塞直到绑定成功
     ChannelFuture channelFuture = b.bind().sync();
     ```

     `b.bind()`方法返回一个端口绑定 Netty 的异步任务`channelFuture`。在这里并没有给`channelFuture`异步任务增加回调监听器，而是阻塞`channelFuture`异步任务，直到端口绑定任务执行完成。

     Netty 中的 IO 操作都会返回异步任务实例（如 channelFuture 实例）。通过该异步任务实例，既可以实现同步阻塞一直到`channelFuture`异步任务执行完成，也可以通过为其增加事件监听器的方式注册异步回调逻辑，以获得 Netty 中的 IO 操作的真正结果。上面使用的是同步阻塞方式。

     至此，服务器正式启动。

7.   自我阻塞，直到监听通道关闭

     ```java
     ChannelFuture closeFuture = channelFuture.channel().closeFuture();
     closeFuture.sync();
     ```

8.   关闭`EventLoopGroup`

     ```java
     // 释放所有资源，包括创建的反应器线程
     workerLoopGroup.shutdownGracefully();
     bossLoopGroup.shutdownGracefully();
     ```

     关闭反应器轮询组，同时会关闭内部的子反应器线程，也会关闭内部的选择器，内部的轮询线程以及负责查询的所有子通道。在子通道关闭后，会释放掉底层的资源，如 Socket 文件描述符等。

### 3.4 ChannelOption

无论是对于`NioServerSocketChannel`父通道类型还是对于`NioSocketChannel`子通道类型，都可以设置一系列的`ChannelOption`。

#### 1.SO_RCVBUF、SO_SNDBUF

这两个为 TCP 传输选项，每个 TCP socket 在内核中都有一个发送缓冲区和接收缓冲区，这两个选项就是用来设置 TCP 连接的两个缓冲区大小的。TCP 的全双工工作模式以及 TCP 的滑动窗口对两个独立的缓冲区都有依赖。

#### 2.TCP_NODELAY

此为 TCP 传输选项，如果设置为 true 就表示立即发送数据。

`TCP_NODELAY`用于开启或关闭 Nagle 算法。如果要求高实时性，有数据发送时就马上发送，就将该选项设置为 true（关闭 Nagle 算法）；如果要减少发送次数、减少网络交互，就设置为 false（开启 Nagle 算法），等累计一定大小的数据后再发送。

>   **说明**
>
>   Nagle 算法将小的碎片数据连接成更大报文（或数据包）来最小化所发送报文的数量，如果需要发送一些较小的报文，则需要禁用该算法。Netty 默认为 true，即报文会立即发送出去，而操作系统默认 false。

#### 3.SO_KEEPALIVE

此为 TCP 传输选项，表示是否开启 TCP 的心跳机制。true 为连接保持心跳，默认为 false。启用时，TCP 会主动探测空闲连接的有效性。默认的心跳间隔是 7200 秒，即 2 小时，Netty 默认关闭该功能。

#### 4.SO_REUSEADDR

此为 TCP 传输选项，为 true 时表示地址复用，默认值为 false，有四种情况需要用到这个参数：

-   当有一个地址和端口相同的连接 socket1 处于 TIME_WAIT 状态时，而又希望启动一个新的连接 socker2 要占用该地址和端口
-   有多块网卡或用 IP Alias 技术的机器在同一端口启动多个进程，但每个进程绑定的本地 IP 地址不能相同
-   同一进程绑定相同的端口到多个 socket 上，但每个 socket 绑定的 IP 地址不同
-   完全相同的地址和端口的重复绑定，但这只用于 UDP 的多播，不用于 TCP

#### 5.SO_LINGER

此为 TCP 传输选项，可以用来控制`socket.close()`方法被调用后的行为，包括延迟关闭事件。

-   如果此选项设置为 -1，就表示`socket.close()`方法在调用后立即返回，但操作系统底层会将发送缓冲区的数据全部发送到对端
-   如果设置为 0，就表示`socket.close()`方法在调用后会立即返回，但是操作系统会放弃发送缓冲区数据，直接向对端发送 RST 包，对端将收到复位错误
-   如果设置为非 0 整数值，就表示调用`socket.close()`方法的线程被阻塞，直到延迟时间到来，发送缓冲区的数据发送完毕，若超时，则对端会收到复位错误

默认值为 -1，表示禁用该功能。

#### 6.SO_BACKLOG

此为 TCP 传输选项，表示服务端接收连接的队列长度，如果队列已满，客户端连接将被拒绝。服务端在处理客户端新连接请求时（三次握手）是**顺序处理**的，同一时间只能处理一个客户端请求，当多个客户端到来时，服务端将不能处理的客户端连接请求放在队列中等待，队列的大小通过`SO_BACKLOG`指定。

具体来说，服务端对完成第二次握手的连接放在一个队列（暂称 A 队列），如果进一步完成第三次握手，再把连接从 A 队列移动到新队列（暂称 B 队列），接下来应用程序会通过调用`accept()`方法取出握手成功的连接，而系统则会将该连接从 B 队列移除。A 和 B 队列的长度之和是`SO_BACKLOG`的值。当 A 和 B 队列的长度之和大于`SO_BACKLOG`时，新连接将被 TCP 内核拒绝。所以如果`SO_BACKLOG`过小，accept 速度可能会跟不上，A 和 B 队列全满，导致新客户端无法连接。

>   **说明**
>
>   `SO_BACKLOG`对程序支持的连接数并无影响，影响的只是还没有被 accept 取出的连接数，也就是三次握手的排队连接数。
>
>   如果连接建立频繁，服务器处理新连接较慢，那么可以适当调大这个参数。

## 4.详解Channel

通道代表网络连接，由他负责同对端进行网络通信，既可以写入数据到对端，也可以从对端读取数据。

### 4.1 主要成员和方法

Netty 通道的抽象类`AbstractChannel`的构造函数如下：

```java
protected AbstractChannel(Channel parent) {
  this.parent = parent;
  id = newId();
  // 新建一个底层的 NIO 通道，完成实际的 IO 操作
  unsafe = newUnsafe();
  pipeline = newChannelPipeline();
}
```

#### 1.成员

-   `pipeline`

    表示处理器的流水线，通道进行初始化的时候，pipeline 属性会被初始化为`DefaultChannelPipeline`的实例，每个通道都拥有一条处理器流水线。

-   `parent`

-   表示父通道属性，对于连接监听通道（如 NioServerSocketChannel）其 parent 属性为 null；对于传输通道（如 NioSocketChannel）其 parent 属性为接收到该连接的监听通道。

几乎所有的 Netty 通道实现类都继承了`AbstractChannel`抽象类，都拥有上面的 parent 和 pipeline 两个属性成员。

#### 2.方法

-   `ChannelFuture connect(SocketAddress address)`

    连接远程服务器。传入远程服务器的地址，调用后会立即返回一个执行连接操作的异步任务`ChannelFuture`。此方法在客户端的传输通道使用。

-   `ChannelFuture bind(SocketAddress address)`

    绑定监听地址，开始监听新的客户端连接。此方法在服务器的新连接监听和接收通道时调用。

-   `ChannelFuture close()`

    关闭通道连接，返回连接关闭的`ChannelFuture`异步任务。如果需要在连接正式关闭后执行其他操作，则需要为异步任务设置回调方法；或者调用`ChannelFuture`异步任务的`sync()`方法来阻塞当前线程，一直等到通道关闭的异步任务执行完毕。

-   `Channel read()`

    读取通道数据，并且启动入站处理。具体来说，从内部的 Java NIO Channel 通道读取数据，然后启动内部的 Pipeline 流水线，开启数据读取的入站处理。此方法的返回通道自身用于链式调用。

-   `ChannelFuture write(Object o)`

    启程出站流水处理，把处理后的最终数据写到底层通道，返回出站处理的异步处理任务。

-   `Channel flush()`

    将缓冲区的数据立即写出到对端。调用前面的`write()`出站处理时，并不能将数据直接写出到对端，`write`操作的作用在大部分情况下仅仅是写入操作系统的缓冲区，操作系统会根据缓冲区的情况决定何时把数据写到对端。执行`flush()`方法会立即将缓冲区的数据写到对端。

以上 6 种方法仅仅是比较常见的通道方法。

### 4.2 EmbeddedChannel

模拟入站和出站的操作，底层不进行实际传输，不需要启动 Netty 服务器和客户端。除了不进行传输之外，EnbeddedChannel 的其他事件机制和处理流程和真正的传输通道一样。方便开发人员在单元测试用例中快速的进行`ChannelHandler`业务处理器的单元测试。

辅助方法

| 名称                  | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| **`writeInbound()`**  | 向通道写入入站数据，模拟真实通道收到数据的场景，写入的数据会被流水线上的入站处理器处理 |
| `readInbound()`       | 从 EmbeddedChannel 中读取入站数据，返回经过流水线最后一个入站处理器处理完成之后的入站数据。如果没有数据则返回 null |
| **`writeOutbound()`** | 向通道写入出站数据，模拟真实通道发送数据，写入的数据会被流水线上的出站处理器处理 |
| `readOutbound()`      | 从 EmbeddedChannel 中读取出站数据，返回经过流水线最后一个出站处理器处理之后的出站数据。如果没有数据则返回 null |
| `finish()`            | 结束 EmbeddedChannel，会调用通道的`close()`方法              |

## 5.详解Handler

在 Reactor 经典模式中，反应器查询到 IO 事件后会分发到 handler 业务处理器，由 Handler 完成 IO 操作和业务处理。

整个 IO 处理操作环节大致包括：从通道读取数据包、数据包解码、业务处理、目标数据编码、把数据包写到通道，然后由通道发送到对端，如下图：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302042328883.png" alt="image-20230204232848855" style="zoom:50%;" />

整个 IO 处理操作环节中，从通道读数据包和由通道发送到对端这两个过程由 Netty 的底层负责，用户程序主要涉及的 Handler 环节为数据包解码、业务处理、目标数据编码、把数据包写到通道中。

从开发人员角度看入站和出站两种类型操作特点为：

-   入站处理出发的方向为自底向上，从 Netty 的内部（如通道）到`ChannelInboundHandler`入站处理器
-   出站处理出发的方向为自顶向下，从`ChannelOutboundHandler`出站处理器到 Netty 的内部

从这个角度看，IO 处理操作环节前面的数据包解码、业务处理两个环节属于入站处理器的工作；后面的目标数据编码、把数据包写到通道这两个环节属于出站处理器的工作。

### 5.1 ChannelInboundHandler

当对端数据入站到 Netty 通道时，Netty 将触发`ChannelInboundHandler`入站处理器对应的 API，进行入站操作处理。

核心方法有：

-   `channelRegistered()`

    当通道注册完成后，Netty 会调用`fireChannelRegistered()`触发通道注册事件，而在通道流水线注册过的入站处理器的`channelRegistered()`回调方法会被调用。

-   `channelActive()`

    当通道激活完成后，Netty 会调用`fireChannelActive()`触发通道激活事件，而在通道流水线注册过的入站处理器的`channelActive()`回调方法会被调用。

-   `channelRead()`

    当通道缓冲区可读时，Netty 会调用`fireChannelRead()`触发通道可读事件，而在通道流水线注册过的入站处理器的`channelRead()`回调方法会被调用，以便完成入站数据的读取和处理。

-   `channelReadComplete()`

    当通道缓冲区读完时，Netty 会调用`fireChannelReadComplete()`触发通道缓冲区读完事件，而在通道流水线注册过的入站处理器的`channelReadComplete()`回调方法会被调用。

-   `channelinactive()`

    当连接被断开或不可用时，Netty 会调用`fireChannelInactive()`触发连接不可用事件，而在通道流水线注册过的入站处理器的`channelInactive()`回调方法会被调用。

-   `exceptionCaught()`

    当通道处理过程发生异常时，Netty 会调用`fireExceptionCaught()`触发异常捕获事件，而在通道流水线注册过的入站处理器的`exceptionCaught()`回调方法会被调用。这个方法是在`Channelhandler`中定义的方法，入站处理器、出站处理器接口都继承了该方法。

在 Netty 中，入站处理器的默认实现为`ChannelInboundHandlerAdapter`，在实际开发中只需要继承该类，重写需要的回调方法即可。

### 5.2 ChannelOutboundHandler

当业务处理完成后，Netty 会调用一系列的`ChannelOutboundHandler`出站处理器完成 Netty 通道到底层通道的操作，比如建立底层连接、断开底层连接、写入底层 Java NIO 通道等。

Netty 出站处理的方向是通过上层 Netty 通道去操作底层 Java IO 通道，主要方法有：

-   `bind()`

    监听地址（IP + 端口）绑定：完成底层 Java IO 通道的 IP 地址绑定，如果使用 TCP 传输协议，这个方法将用于服务端。

-   `connect()`

    连接服务端：完成底层 Java IO 通道的服务端的连接操作。如果使用 TCP 传输协议，这个方法将用于服务端。

-   `write()`

    写数据到底层：完成 Netty 通道向底层 Java IO 通道的数据写入操作。此方法仅仅是触发一下操作，并不是完成实际的数据写入操作。

-   `flush()`

    将底层缓存区的数据腾空，立即写出到对端。

-   `read()`

    从底层读数据：完成 Netty 通道从 Java IO 通道的数据读取。

-   `disConnect()`

    断开服务器连接：断开底层 Java IO 通道的 socket 连接。如果使用 TCP 传输协议，此方法主要用于客户端。

-   `close()`

    主动关闭通道：关闭底层的通道，例如服务端的新连接监听通道。

在 Netty 中，入站处理器的默认实现为`ChannelOutboundHandlerAdapter`，在实际开发中只需要继承该类，重写需要的回调方法即可。

### 5.3 ChannelInitializer

如果需要向流水线中装配 Handler，就需要借助通道的初始化处理器——`ChannelInitializer`。

例如 NettyDiscardServer 中调用了`childHandler()`设置了一个`ChannelInitializer`实例：

```java
b.childHandler(new ChannelInitializer<SocketChannel>() {
  @Override
  protected void initChannel(SocketChannel ch) throws Exception {
    ch.pipeline().addLast(new NettyDiscardHandler());
  }
});
```

在通道初始化时，会调用提前注册的初始化处理器的`initChannel()`方法。例如，在父通道接受到新连接并且要初始化其子通道时，会调用初始化器的`initChannel()`方法，并且会将新接受的通道作为参数，传递给此方法。因此一般用于向流水线中装配 Handler。

### 5.4 案例：ChannelInboundHandler生命周期演示

定义一个入站 Handler，继承`ChannelInbundHandlerAdapter`适配器，实现大部分方法

```java
@ChannelHandler.Sharable
public class InHandlerDemo extends ChannelInboundHandlerAdapter {
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        Logger.info("被调用：handlerAdded()");
        super.handlerAdded(ctx);
    }
    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        Logger.info("被调用：handlerRemoved()");
        super.handlerAdded(ctx);
    }

    @Override
    public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        Logger.info("被调用：channelRegistered()");
        super.channelRegistered(ctx);
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        Logger.info("被调用：channelActive()");
        super.channelActive(ctx);
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        Logger.info("被调用：channelInactive()");
        super.channelInactive(ctx);
    }

    @Override
    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
        Logger.info("被调用: channelUnregistered()");
        super.channelUnregistered(ctx);
    }


    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        Logger.info("被调用：channelRead()");
        super.channelRead(ctx, msg);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        Logger.info("被调用：channelReadComplete()");
        super.channelReadComplete(ctx);
    }

}
```

除了`channelRead()`和`channelReadComplete()`是入站处理方法，其他的 6 个方法都是入站处理器的周期方法。

接下来编写测试类

```java
public class InHandlerDemoTester {
  
  @Test
  public void testInHandlerProcessWithChannelInitializer() {
    final InHandlerDemo inHandler = new InHandlerDemo();
    //初始化处理器
    ChannelInitializer initializer = new ChannelInitializer<EmbeddedChannel>() {
      protected void initChannel(EmbeddedChannel ch) {
        ch.pipeline().addLast(inHandler);
        ch.pipeline().addLast(new LoggingHandler(LogLevel.DEBUG));
      }
    };
    //创建嵌入式通道
    EmbeddedChannel channel = new EmbeddedChannel(initializer);
    ByteBuf buf = Unpooled.buffer();
    buf.writeInt(1);
    //模拟入站，写一个入站包
    channel.writeInbound(buf);
    channel.flush();
    //模拟入站，再写一个入站包
    channel.writeInbound(buf);
    channel.flush();
    //通道关闭
    channel.close();
    try {
      Thread.sleep(Integer.MAX_VALUE);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }

}
```

也可以不使用`Channelinitializer`，创建嵌入式通道时直接传入处理器

```java
final InHandlerDemo inHandler = new InHandlerDemo();
//创建嵌入式通道
EmbeddedChannel channel = new EmbeddedChannel(inHandler,new LoggingHandler(LogLevel.DEBUG));
```

结果

```bash
信息: 被调用：handlerAdded()
信息: 被调用：channelRegistered()
信息: 被调用：channelActive()
信息: 被调用：channelRead()
信息: 被调用：channelReadComplete()
信息: 被调用：channelRead()
信息: 被调用：channelReadComplete()
信息: 被调用：channelInactive()
信息: 被调用: channelUnregistered()
信息: 被调用：handlerRemoved()
```

可以发现，回调方法的执行顺序为：

```java
handlerAdded() - channelRegistered() - channelActive() - 数据传输的入站回调 - channelInactive() - channelUnregisterd() - handlerRemoved()
```

其中，数据传输的入站回调过程为：

```java
channelRead() - channelReadComplete()
```

读数据的入站回调过程会随着入站数据的数量被重复调用，如果没有数据则不会被调用：

-   `channelRead()`

    有数据包入站，通道可读。入站处理器的此方法会被依次从前往后调用。

-   `channelReadComplete()`

    流水线完成入站处理后，会从前向后依次回调每个入站处理器的此方法，表示数据读取完毕。

其余的 6 个方法都和 Channelhandler 的生命周期有关，不论有没有数据都会被调用：

-   `handlerAdded()`

    当 Handler 被加入到 Pipeline 后，此方法将被回调，即在完成`ch.pipeline().addLast(handler)`语句后会回调`handlerAdded()`。

-   `channelRegistered()`

    当通道成功绑定一个`NioEventLoop`反应器后，此方法将被回调。

-   `channelActive()`

    当通道激活成功后，此方法将被回调。

    通道激活成功指的是所有的业务处理器添加、注册的异步任务完成，并且与`NioEventLoop`反应器绑定的异步任务完成。

-   `channelInactive()`

    当通道的底层连接已经不是`ESTABLISH`状态或者底层连接已经关闭，会首先回调所有业务处理器的此方法。

-   `channelUnregistered()`

    通道和`NioEventLoop`反应器解除绑定，移除掉对这条通道的事件处理之后，回调所有业务处理器的此方法。

-   `handlerRemoved()`

    Netty 会移除掉通道上所有的业务处理器，并且回调所有业务处理器的此方法。

### 5.5 案例：ChannelOutboundHandler生命周期演示

编写一个类继承`ChannelOutboundHandlerAdapter`并重写大部分方法。

```java
@ChannelHandler.Sharable
  public class OutHandlerDemo extends ChannelOutboundHandlerAdapter {

    private Logger logger = Logger.getLogger("outHandlerDemo");

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
      logger.info("被调用：handlerAdded()");
      super.handlerAdded(ctx);
    }

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
      logger.info("被调用： handlerRemoved()");
      super.handlerRemoved(ctx);
    }

    @Override
    public void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception {
      logger.info("被调用： bind()");
      super.bind(ctx, localAddress, promise);
    }

    @Override
    public void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress, SocketAddress localAddress,
                        ChannelPromise promise) throws Exception {
      logger.info("被调用： connect()");
      super.connect(ctx, remoteAddress, localAddress, promise);
    }

    @Override
    public void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
      logger.info("被调用： disconnect()");
      super.disconnect(ctx, promise);
    }

    @Override
    public void read(ChannelHandlerContext ctx) throws Exception {
      logger.info("被调用： read()");
      super.read(ctx);
    }

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
      logger.info("被调用： write()");
      super.write(ctx, msg, promise);
    }

    @Override
    public void flush(ChannelHandlerContext ctx) throws Exception {
      logger.info("被调用： flush()");
      super.flush(ctx);
    }
  }
```

测试代码

```java
public class OutHandlerDemoTester {

  @Test
  public void testlifeCircle() {
    final OutHandlerDemo outHandler = new OutHandlerDemo();
    ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
      @Override
      protected void initChannel(EmbeddedChannel ch) {
        ch.pipeline().addLast(outHandler);
      }
    };

    EmbeddedChannel channel = new EmbeddedChannel(i);
    ByteBuf buf = Unpooled.buffer();
    buf.writeInt(1);
    // 出站时，可以使用以下两种写入刷新方法，都会返回 ChannelFuture 类型
    ChannelFuture f1 = channel.pipeline().writeAndFlush(buf);
    ChannelFuture f2 = channel.writeAndFlush(buf);

    f1.addListener(future -> {
      if (future.isSuccess()) {
        System.out.println("write for f1 is finished");
      }
    });
    f2.addListener(future -> {
      if (future.isSuccess()) {
        System.out.println("write for f2 is finished");
      }
    });

    if (f1.isSuccess() && f2.isSuccess()) {
      channel.close();
    }

    try {
      Thread.sleep(Integer.MAX_VALUE);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }

  }

}
```

结果

```bash
信息: 被调用：handlerAdded()
信息: 被调用： read()
信息: 被调用： write()
信息: 被调用： flush()
信息: 被调用： write()
信息: 被调用： flush()
write for f1 is finished
write for f2 is finished
信息: 被调用： handlerRemoved()
```

可以发现回调顺序为：

```java
handlerAdded() - read() - write() - flush - handlerRemoved()
```

其中`handlerAdded()`、`read()`与`handlerRemoved()`是与生命周期有关的方法。

>   **`Channel`的`writeOutbound()`和`writeAndFlush()`的区别**
>
>   两者都可以向通道写入并刷新数据，但是`channel.writeAndFlush()`会返回`ChannelFuture`对象，可以用来添加监听器。而`channel.writeOutbound()`返回的是一个`boolean`值。

## 6.详解Pipeline

一条 Netty 通道需要很多业务处理器来处理业务，每条通道内部都有一条 Pipeline 将 handler 装配起来。Netty 的`ChannelPipeline`是基于责任链设计模式来设计的，内部是一个双向链表结构，能够支持动态地添加和删除业务处理器。

### 6.1 Pipeline入站处理流程

为了完整演示 Pipeline 入站处理流程，新建三个简单的入站处理器：SimpleInHandlerA、SimpleInHandlerB、SimpleInHandlerC。通过`ChannelInitializer`将它们依次添加到流水线中。

```java
static class SimpleInHandlerA extends ChannelInboundHandlerAdapter {
  @Override
  public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    Logger.info("入站处理器 A: 被回调 ");
    super.channelRead(ctx, msg);
    // 也可以调用 ctx.fireChannelRead(msg);
  }

  @Override
  public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
    Logger.info("入站处理器 A: 被回调 ");
    super.channelReadComplete(ctx); // 入站操作的传播
    // 也可以调用 ctx.fireChannelReadComplete();
  }


}
static class SimpleInHandlerB extends ChannelInboundHandlerAdapter {
  @Override
  public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    Logger.info("入站处理器 B: 被回调 ");
    super.channelRead(ctx, msg);
    // 也可以调用 ctx.fireChannelRead(msg);
  }

  @Override
  public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
    Logger.info("入站处理器 B: 被回调 ");
    super.channelReadComplete(ctx); // 入站操作的传播
    // 也可以调用 ctx.fireChannelReadComplete();
  }
}


static class SimpleInHandlerC extends ChannelInboundHandlerAdapter {
  @Override
  public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    Logger.info("入站处理器 C: 被回调 ");
    super.channelRead(ctx, msg);
    // 也可以调用 ctx.fireChannelRead(msg);
  }

  @Override
  public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
    Logger.info("入站处理器 C: 被回调 ");
    super.channelReadComplete(ctx); // 入站操作的传播
    // 也可以调用 ctx.fireChannelReadComplete();
  }
}


@Test
public void testPipelineInBound() {
  ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
    protected void initChannel(EmbeddedChannel ch) {
      ch.pipeline().addLast(new LoggingHandler(LogLevel.DEBUG));
      ch.pipeline().addLast(new SimpleInHandlerA());
      ch.pipeline().addLast(new SimpleInHandlerB());
      ch.pipeline().addLast(new SimpleInHandlerC());

    }
  };
  EmbeddedChannel channel = new EmbeddedChannel(i);
  ByteBuf buf = Unpooled.buffer();
  buf.writeInt(1);
  // 向通道写一个入站报文
  channel.writeInbound(buf);
  ThreadUtil.sleepSeconds(Integer.MAX_VALUE);
}
```

在以上三个内部入站处理器的`channelRead()`方法中，我们打印当前 Handler 业务处理器的信息，然后调用了父类的`channelRead()`方法，调用父类方法的主要作用是把当前入站处理器中处理完毕的结果传递到下一个入站处理器。这里也可以调用`ctx.fireChannelRead(msg)`将结果传递给	

>   **提示**
>
>   在入站处理时，大致可通过两种方法将数据传递给下一站：
>
>   -   `super.channelXxx(ChannelhandlerContext)`
>   -   `ctx.fireChannelXxx()`
>
>   父类`ChannelInboundHandlerAdapter`的`channelRead()`方法默认实现就是是通过调用`ctx.fireChannelRead(msg)`实现的。

输出的结果如下：

```bash
二月 05, 2023 2:49:55 上午 InPipeline$SimpleInHandlerA channelRead
信息: 入站处理器 A: 被回调 
二月 05, 2023 2:49:55 上午 InPipeline$SimpleInHandlerB channelRead
信息: 入站处理器 B: 被回调 
二月 05, 2023 2:49:55 上午 InPipeline$SimpleInHandlerC channelRead
信息: 入站处理器 C: 被回调 
二月 05, 2023 2:49:55 上午 InPipeline$SimpleInHandlerA channelReadComplete
信息: 入站处理器 A: 被回调 
二月 05, 2023 2:49:55 上午 InPipeline$SimpleInHandlerB channelReadComplete
信息: 入站处理器 B: 被回调 
二月 05, 2023 2:49:55 上午 InPipeline$SimpleInHandlerC channelReadComplete
信息: 入站处理器 C: 被回调 
```

### 6.2 Pipeline 出站处理流程

类似的，创建三个简单的出站处理器：SimpleOutHandlerA、SimpleOutHandlerB、SimpleOutHandlerC，通过`ChannelInitializer`将它们依次添加到流水线中。

```java
public class OutPipeline {

  private static Logger logger = Logger.getLogger("inPipeline");

  static class SimpleOutHandlerA extends ChannelOutboundHandlerAdapter {

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
      logger.info("出站处理器 A: 被回调");
      super.write(ctx, msg, promise);
    }

  }

  static class SimpleOutHandlerB extends ChannelOutboundHandlerAdapter {

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
      logger.info("出站处理器 B: 被回调");
      super.write(ctx, msg, promise);
    }

  }

  static class SimpleOutHandlerC extends ChannelOutboundHandlerAdapter {

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
      logger.info("出站处理器 C: 被回调");
      super.write(ctx, msg, promise);
    }

  }

  @Test
  public void testPipelineOutbound() {
    ChannelInitializer<EmbeddedChannel> initializer = new ChannelInitializer<EmbeddedChannel>() {

      @Override
      protected void initChannel(EmbeddedChannel ch) throws Exception {
        ch.pipeline().addLast(new SimpleOutHandlerA());
        ch.pipeline().addLast(new SimpleOutHandlerB());
        ch.pipeline().addLast(new SimpleOutHandlerC());
      }

    };

    EmbeddedChannel channel = new EmbeddedChannel(initializer);
    ByteBuf buf = Unpooled.buffer();
    buf.writeInt(1);
    channel.writeAndFlush(buf);
    
    try {
      Thread.sleep(Integer.MAX_VALUE);
    } catch (InterruptedException e) {
      // TODO Auto-generated catch block
      e.printStackTrace();
    }
  }

}
```

输出结果为：

```bash
二月 05, 2023 2:47:34 下午 OutPipeline$SimpleOutHandlerC write
信息: 出站处理器 C: 被回调
二月 05, 2023 2:47:34 下午 OutPipeline$SimpleOutHandlerB write
信息: 出站处理器 B: 被回调
二月 05, 2023 2:47:34 下午 OutPipeline$SimpleOutHandlerA write
信息: 出站处理器 A: 被回调
```

可以看到，出站处理器的执行顺序与添加顺序相反：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302051448380.png" alt="image-20230205144824352" style="zoom:50%;" />

>   **提示**
>
>   在出站处理时，大致可通过两种方法将数据传递给下一站：
>
>   -   `super.write(ctx, msg, promise);`
>   -   `ctx.write(msg, promise)`
>
>   `ChannelInboundHandlerAdapter`的`write()`方法默认实现就是通过调用`ctx.write(msg)`实现的。

### 6.3 ChannelHandlerContext

在 Netty 的设计中 Handler 是无状态的，不保存 Channel 相关信息，而 Pipeline 是有状态的，保存了 Channel 的关系，Netty 创建了`ChannelHandlerContext`来管理这种关系。

不论定义的是哪种类型的业务处理器，最终都是已双向链表的形式保存在流水线中。这里流水线的节点类型是业务处理器的包装类型`ChannelHandlerContext`。

当业务处理器 Handler 被添加到流水线中时会为其专门创建一个`ChannelhandlerContext`实例，主要封装了`ChannelHandler`和`ChannelPipeline`之间的关联关系。

所以，流水线`ChannelPipeline`中的双向链接实质是一个由``ChannelHandlerContext`组成的双向链表。

`ChannelPipeline`流水线的示意图大致如下：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302051523593.png" alt="image-20230205152333548" style="zoom:50%;" />

`ChannelHandlerContext`中的方法可以分为两类：第一类是获取上下文所关联的 Netty 组件实例，如关联的通道、关联的流水线、上下文内部 Handler 业务处理器实例等；第二类是入站出站处理方法。

`Channel`、`ChannelPipeline`、`ChannelHandlerContext`中入站出站处理方法的不同之处在于：

-   如果通过`Channel`或者`ChannelPipeline`的实例来调用入站出站方法，他们就会在整条流水线中传播
-   如果是通过`ChannelHandlerContext`来调用，就只会从当前的节点开始往同类型的下一站处理器传播，而不是在整条流水线从头至尾进行完整的传播

具体案例在<a href="#6.6 截断流程">6.6 截断流程</a>

### 6.4 HeadContext与TailContext

通道流水线默认装配了两个处理器上下文：头部上下文`HeadContext`和尾部上下文`TailContext`。我们每次添加的处理器上下文节点都添加在`HeadContext`和`TailContext`之间。

例如，在添加了一些必要的解码器、业务处理器、编码器之后，一条流水线的结构大致如下：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302051550005.png" alt="image-20230205155025960" style="zoom:50%;" />

流水线尾部的`TailContext`不仅仅是一个上下文类，还是一个入站处理器类，实现了所有入站处理回调方法，主要进行一些收尾工作，如释放缓冲区对象、完成异常处理等。

`TailContext`是流水线默认实现类`DefaultChannelPipeline`的一个内部类，代码大致如下：

```java
public class DefaultChannelPipeline implements ChannelPipeline {
  // 内部类：尾部处理器和尾部上下文是同一个类
  final class TailContext extends AbstractChannelhandlerContext implements ChannelInboundHandler {
    // 入站处理方法：读取通道
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
      // 释放缓冲区
    }
    // 省略其他入站处理方法
  }
}
```

流水线头部的`HeadContext`比`TailContext`复杂得多，既是一个出站处理器，也是一个入站处理器，还保存了一个`unsafe`（完成实际通道传输的类）实例，即还需要负责最终的通道传输工作。

`HeadContext`也是流水线默认实现类`DefaultChannelPipeline`的一个内部类，代码大致如下：

```java
public class DefaultChannelPipeline implements ChannelPipeline {
  final class HeadContext extends AbstractChannelHandlerContext implements ChannelOutboundHandler, ChannelInboundHandler {
    // 传输操作类实例：完成通道最终的输入、输出等操作
    // 此类专供 Netty 内部使用，应用程序不能使用
    private final Unsafe unsafe;
    
    // 入站处理举例：入站（从 Channel 到 Handler）读操作
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
      ctx.fireChannelRead(msg);
    }
    
    // 出站处理举例：出站（从 Handler 到 Channel）读取传输数据
    @Override
    public void read(ChannelHandlerContext ctx) {
      unsafe.beginRead();
    }
    
    // 出站处理举例：出站写操作
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
      unsafe.write(msg, promise);
    }
    
    // 省略其他处理方法
  }
}
```

### 6.5 入站和出站的双向链接操作

下面摘取流水线的一个入站（读）操作和一个出站（写）操作：

```java
final class DefaultChannelPipeline implements ChannelPipeline {
  
  final AbstractChannelHandlerContext head;
  final AbstractChannelHandlerContext tail;
  
  // 启动流水线的出站写
  @Override
  public ChannelFuture write(Object msg) {
    return tail.write(msg); // 从后往前传递
  }
  
  // 启动流水线的入站读
  @Override
  public ChannelPipeline fireChannelRead(Object msg) {
    head.fireChannelRead(msg); // 从头往后传递
    return this;
  }
}
```

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302051550005.png" alt="image-20230205155025960" style="zoom:50%;" />

#### 1.入站操作

流水线的入站流程是从`fireXxx()`方法开始的。在`fireChannelRead()`的源码中，从流水线的头节点 Head 开始，将入站的 msg 数据沿着流水线上的入站处理器逐个向后传递。

如果所有的入站处理过程都没有截断流水线的处理，则该入站数据 msg（如 ByteBuffer 缓冲区）将一直传递到流水线的末尾，也就是`TailContext`。

接下来查看下入站操作的传播行为，下面是`fireChannelRead()`方法的源码：

```java
abstract class AbstractChannelHandlerContext implements ChannelHandlerContext {
  @Override
  public ChannelHandlerContext fireChannelRead(final Object msg) {
    if (msg == null) {
      throw new NullPointerException("msg");
    }
    // 在双向链表中向后查找，找到下一个入站节点（后继的入站节点）
    final AbstractChannelHandlerContext next = findContextInbound();
    // 获取后继的处理线程
    EventExecutor executor = new.executor();
    // 下一个 Handler 节点的 EventLoop 会否和当前 Handler 节点的是同一个
    if (executor.inEventLoop()) {
      // 执行后继上下文所包装的处理器
      next.invokeChannelRead(msg);
    } else {
      // 如果当前线程不是后继的处理线程，则提交到后继节点的 EventLoop 中处理线程去排队
      // 保障该节点的处理器被设置的线程调用
      executor.execute(new OneTimeTask() {
        @Override
        public void run() {
          // 提交到后继处理线程
          next.invokeChannelRead(msg);
        }
      });
    }
    return this;
  }
}
```

-   首先在`fireChannelRead()`方法中调用`findContextInbound()`方法找到下一个入站节点（后继的入站节点）

    ```java
    private AbstractChannelHandlerContext findContextInbound() {
      AbstractChannelHandlerContext ctx = this;
      do {
        // 向后查找，一直到末尾或者找到入站类型节点为止
        ctx = ctx.next;
      } while (!ctx.inbound);
      return ctx;
    }
    ```

-   找到下一个入站节点后，准备开始执行下一站所包装的处理器，这里需要确保执行的线程是该 Context 实例（下一个入站节点）的 executor 成员线程以保证线程安全（在添加 Handler 时，可以指定处理该 Handler 的 EventLoop 线程）

-   执行下一站的处理器方法

    ```java
    private void invokeChannelRead(Object msg) {
      try {
        ((ChannelInboundHandler) handler()).channelRead(this, msg);
      } catch (Throwable t) {
        notifyHandlerException(t);
      }
    }
    ```

#### 2.出站流程

而流水线的出站流程是从流水线的尾部节点 Tail 开始，将出战 msg 数据沿着流水线上的出站处理器逐个向前传递。

出站 msg 数据在经过所有出站处理器之后，将一直传递到流水线的头部，也就是`HeadContext`处理器，并且通过`unsafe`传输实例将二进制数据写入底层传输通道，完成整个传输处理过程。

Pipeline 的出站传播除了方向是相反的，其余的地方与入站传播大致相同：

```java
private AbstractChannelhandlerContext findContextOutbound() {
  AbstracxtChannelhandlerContext ctx = this;
  do {
    // 向前查找，直到头部或者找到一个出站 Context 为止
    ctx = ctx.prev;
  } while (!ctx.outbound);
  return ctx;
}
```

#### 3.中间过程

流水线链表的节点实现默认是`AbstractChannelhandlerContext`，此类也是`HeadContext`和`TailContext`的父类。pipeline 内部的双向链表的指针维护以及节点前驱和后继的计算方法都在这个类中实现。其主要成员属性如下：

```java
abstract class AbstractChannelHandlerContext implements ChannelHandlerContext {
  // 双向链表的指针：指向后继
  volatile AbstractChannelHandlerContext next;
  // 双向链表的指针：指向前驱
  volatile AbstractChannelhandlerContext prev;
  
  // 标志：是否为入站节点
  private final boolean inbound;
  // 标志：是否为出站节点
  private final boolean outbound;
  // 上下文节点所关联的通道
  privaet final AbstractChannel channel;
  // 所属流水线
  private final DefaultChannelPipeline pipeline;
  // 上下文节点名称，可在加入流水线时指定
  private final String name;
  // 节点的执行线程，如果没有特别设置，则为通道的 IO 线程
  final EventExecutor executor;
}
```

### 6.6 截断流程

创建三个 InboundHandler 和三个 OutboundHandler，在第三个`SimpleInboundHandlerC`中写入数据，来触发出站处理器

```java
public class InboundOutboundPipelineTest {

  private Logger logger = Logger.getLogger("inboundOutboundPipelineTest");

  class SimpleInboundHandlerA extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
      logger.info("inbound A invoked");
      super.channelRead(ctx, msg);
    }

  }

  class SimpleInboundHandlerB extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
      logger.info("inbound B invoked");
      super.channelRead(ctx, msg);
    }

  }

  class SimpleInboundHandlerC extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
      logger.info("inbound C invoked");
      ctx.channel().write(msg);
      // ctx.channel().pipeline().write(msg);
    }

  }

  class SimpleOutboundHandlerA extends ChannelOutboundHandlerAdapter {

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
      logger.info("outbound A invoked");
      super.write(ctx, msg, promise);
    }

  }

  class SimpleOutboundHandlerB extends ChannelOutboundHandlerAdapter {

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
      logger.info("outbound B invoked");
      super.write(ctx, msg, promise);
    }

  }

  class SimpleOutboundHandlerC extends ChannelOutboundHandlerAdapter {

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
      logger.info("outbound C invoked");
      super.write(ctx, msg, promise);
    }

  }

  @Test
  void inboundOutboundTest() {
    ChannelInitializer<EmbeddedChannel> initializer = new ChannelInitializer<EmbeddedChannel>() {

      @Override
      protected void initChannel(EmbeddedChannel ch) throws Exception {
        ch.pipeline().addLast(new SimpleInboundHandlerA());
        ch.pipeline().addLast(new SimpleInboundHandlerB());
        ch.pipeline().addLast(new SimpleInboundHandlerC());
        ch.pipeline().addLast(new SimpleOutboundHandlerA());
        ch.pipeline().addLast(new SimpleOutboundHandlerB());
        ch.pipeline().addLast(new SimpleOutboundHandlerC());
      }

    };
    EmbeddedChannel channel = new EmbeddedChannel(initializer);
    ByteBuf buf = Unpooled.buffer();
    buf.writeInt(1);
    channel.writeInbound(buf);

    try {
      Thread.sleep(Integer.MAX_VALUE);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }

}
```

输出结果

```bash
信息: inbound A invoked
信息: inbound B invoked
信息: inbound C invoked
信息: outbound C invoked
信息: outbound B invoked
信息: outbound A invoked
```

处理器之间的联系为：

```bash
HeadContext <-> inboundA <-> inboundB <-> inboundC <-> outboundA <-> outboundB <-> outboundC <-> TailContext
```

**几个重要方法的区别**

-   `super.channelXxx(ChannelhandlerContext)`或`ctx.fireChannelXxx()`的作用是**将数据传递给下一站直到 TailContext**，如果注释掉`InboundHandlerB`中的`super.channelRead(ctx, msg)`，则会在`InboundHandlerB`处中断：

    ```bash
    信息: inbound A invoked
    信息: inbound B invoked
    ```

-   `ctx.write(msg, promise)`作用是**传递给当前节点的上一个出站处理器知道 HeadContext**，如果注释掉`OutboundHandlerB`中的`super.write(ctx, msg, promise)`，则会在`OutboundHandlerB`处中断：

    ```bash
    信息: inbound A invocked
    信息: inbound B invocked
    信息: inbound C invocked
    信息: outbound C invocked
    信息: outbound B invocked
    ```

    相应的，数据也不会传递到头节点`HeadContext`并通过`unsafe`写入底层通道。

-   `ctx.channel().write(msg)`和`ctx.channel().pipeline().write(msg)`的作用是跳过后续的入站处理器，直接**从尾部开始触发后续出站处理器的执行**

    -   如果在`InboundHandlerC`中不是调用`ctx.channel().write(msg)`，而是调用`ctx.write(msg)`，则因为`InboundHandlerC`的前面没有任何 OutboundHandler，因此不会打印出站处理器的信息。输出为：

        ```bash
        信息: inbound A invoked
        信息: inbound B invoked
        信息: inbound C invoked
        ```

    -   如果在`OutboundHandlerC`中调用`ctx.channel().write(msg)`或者`ctx.channel().pipeline().write(msg)`，则会循环打印`outbound C invoked`，因为`ctx.channel().write(msg)`是从尾部开始查找，结果又是自己，因此陷入死循环

### 6.7 流水线上热插拔handler

Netty 支持动态地增加、删除流水线上的业务处理器。主要的 Handler 热插拔方法声明在`ChannelPipeline`接口中：

```java
public interface ChannelPipeline extends Iterable<Entry<String, ChannelHandler>> {
  // 在流水线头部增加一个业务处理器，名字由 name 指定
  ChannelPipeline addFirst(String name, ChannelHandler handler);
  // 在流水线尾部增加一个业务处理器，名字由 name 指定
  ChannelPipeline addLast(String name, ChannelHandler handler);
  // 在 baseName 处理器的前面增加一个业务处理器，名字由 name 指定
  ChannelPipeline addBefore(String baseName, String name, ChannelHandler handler);
  // 在 baseName 处理器的后面增加一个业务处理器，名字由 name 指定
  ChannelPipeline addAfter(String baseName, String name, ChannelHandler handler);
  // 删除一个业务处理器实例
  ChannelPipeline remove(Channelhandler handler);
  // 删除一个处理器实例
  ChannelHandler remove(String handler);
  // 删除第一个业务处理器
  ChannelHandler removeFirst();
  // 删除最后一个业务处理器
  ChannelHandler removeLast();
}
```

下面示范如何从流水线动态删除一个 Handler：

```java
public class PipelineHotOperateTest {

  Logger logger = Logger.getLogger("pipelineHotOperateTest");

  class SimpleInboundHandlerA extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
      logger.info("inboundHandler A invoked");
      super.channelRead(ctx, msg);
      ctx.pipeline().remove(this);
    }

  }

  class SimpleInboundHandlerB extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
      logger.info("inboundHandler B invoked");
      super.channelRead(ctx, msg);
    }

  }

  class SimpleInboundHandlerC extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
      logger.info("inboundHandler C invoked");
      super.channelRead(ctx, msg);
    }

  }

  @Test
  void testPipelineHotOperating() {
    ChannelInitializer<EmbeddedChannel> initializer = new ChannelInitializer<EmbeddedChannel>() {

      @Override
      protected void initChannel(EmbeddedChannel ch) throws Exception {
        ch.pipeline().addLast(new SimpleInboundHandlerA());
        ch.pipeline().addLast(new SimpleInboundHandlerB());
        ch.pipeline().addLast(new SimpleInboundHandlerC());
      }

    };

    EmbeddedChannel channel = new EmbeddedChannel(initializer);
    ByteBuf buf = Unpooled.buffer();
    buf.writeInt(1);
    channel.writeInbound(buf);
    channel.writeInbound(buf);
    channel.writeInbound(buf);
  }

}
```

结果如下：

```bash
信息: inboundHandler A invoked
信息: inboundHandler B invoked
信息: inboundHandler C invoked
信息: inboundHandler B invoked
信息: inboundHandler C invoked
信息: inboundHandler B invoked
信息: inboundHandler C invoked
```

可以看到，在`SimpleInboundHandlerA`从流水线删除后，在后面的入站流水处理中已经不再调用`SimpleInboundHandlerA`了。

**ChannelInitializer**

查看源码可以看到，`ChannelInitializer`也是继承了`ChannelInboundHandlerAdapter`，即它也是一种 Handler，在注册完成`channelRegistered`回调方法中调用了`ctx.pipeline().remove(this)`将自己从流水线中删除了，所以该处理器仅仅被执行了一次：

```java
public abstract class ChannelInitializer extends ChannelInboundHandlerAdapter {
  // 通道初始化，抽象方法，需要子类实现
  protected abstract void initChannel(Channel var1) throws Exception;
  // 回调方法：加入通道后（注册完成）出发
  public final void channelRegistered(ChannelHandlerContext ctx) {
    // 调用通道初始化实现
    this.initChannel(ctx.channel());
    // 删除通道初始化处理器
    ctx.pipeline().remove(this);
    // 发送注册消息到下一站
    ctx.fireChannelRegistered();
  }
}
```

## 7.详解ByteBuf

Netty 提供了 ByteBuf 缓冲区组件来代替 Java NIO 的 ByteBuffer 缓冲区组件，以便更加快捷和高效地操作内存缓冲区。

### 7.1 ByteBuf 的优势

与 Java NIO 的 ByteBuffer 相比，ByteBuf 的优势如下：

-   Pooling（池化），减少了内存复制和 GC，提升了效率
-   复合缓冲区类型，支持零复制
-   不需要调用`flip()`方法切换读/写模式
-   可扩展性好
-   可自定义缓冲区类型
-   读取和吸入索引分开
-   方法的链式调用
-   可进行引用计数，方便重复使用

### 7.2 ByteBuf的组成部分

ByteBuf 内部是一个字节数组，从逻辑上可分为 4 个部分：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302052046488.png" alt="image-20230205204651432" style="zoom:50%;" />

第一部分是已用字节；第二部分是可读字节，从 ByteBuf 中读取的数据都来自这一部分；第三部分是可写字节，写入 ByteBuf 的数据都会写入这一部分；第四部分是可扩容字节，表示该 ByteBuf 最多还能扩容的大小。

### 7.3 ByteBuf的重要属性

ByteBuf 通过三个整数类型的属性有效的区分可读数据和可写数据的索引，这三个属性定义在 AbstractByteBuf 抽象类，分别是：

-   `readerIndex`（读指针）：指示读取的起始位置，每读取一个字节自动 +1。一旦`readerIndex`和`writerIndex`相等，则表示 ByteBuf 不可读

-   `writerIndex`（写指针）：指示写入的起始位置，每写一个字节自动 +1。一旦`writerIndex`与`capacity()`容量相等，则表示 ByteBuf 不可写。

    >   **注意**
    >
    >   `capacity()`是一个成员方法，不是一个成员属性，表示 ByteBuf 中可以写入的容量，它的值不一定是最大容量值。

-   `maxCapacity`（最大容量）：表示 ByteBuf 可以扩容的最大容量。当向 ByteBuf 写数据的时候，如果容量不足，可以进行扩容。扩容的最大限度由`maxCapacity`来设定，超过`maxCapacity`则会报错

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302052054719.png" alt="image-20230205205446669" style="zoom:50%;" />

### 7.4 ByteBuf的方法

ByteBuf 的方法大致分为三组：

-   **容量系列**

    -   `capacity()`：表示 ByteBuf 的容量，是废弃的字节数、可读字节数和可写字节数之和
    -   `maxCapacity()`：表示 ByteBuf 能够容纳的最大字节数。当向 ByteBuf 中写数据的时候。如果发现容量不足，则进行扩容，直至扩容到`maxCapacity`的上线

-   **写入系列**

    -   `isWritable()`：表示 ByteBuf 是否可写。如果`capacity()`容量大于`writerIndex`指针的位置，则表示可写

        >   **注意**
        >
        >   `isWritable()`返回 false 并不代表不能往 ByteBuf 中写数据了。如果 Netty 发现往 ByteBuf 中写不了数据了，就会自动扩容 ByteBuf。

    -   `writableBytes()`：取得可写入的字节数，它的值等于`capacity()-writerIndex`
    -   `maxWritableBytes()`：取得最大的可写字节数，它的值等于`maxCalacity-writerIndex`
    -   `writeBytes(byte[] src)`：把入参 src 字节数组中的全部数据写到 ByteBuf
    -   `writeTYPE(TYPE value)`：写入基础数据类型的数据。这里包含了八种基础数据类型：`writeByte()`、`writeBooealn()`、`writeChar()`、`writeShort()`、`writeInt()`、`writeLong()`、`writeFloat()`、`writeDouble()`
    -   `setTYPE(TYPE value)`：设置基础数类型的数据，不同于`writeType()`方法，`setType()`不会改变`writerIndex`指针值
    -   `markWriterIndex()`与`resetWriterIndex()`：前一个方法表示把当前的写指针`writerIndex`属性值保存在`markedWriterIndex`标记属性中；后一个方法表示把之前保存的`markedWriterIndex`的值恢复到写指针`writerIndex`属性中。`markedWriterIndex`相当于一个写指针的暂存属性

-   **读取系列**

    -   `isReadable()`：返回 ByteBuf 是否可读。如果`writerIndex`指针的值大于`readerIndex`指针的值，则表示可读
    -   `readableBytes()`：返回表示 ByteBuf 当前可读取的字节数，它的值等于`writerIndex-readerIndex`
    -   `readBytes(byte[] dst)`：将数据从 BYteBuf 读取到 dst 目标字节数组中，这里 dst 字节数组的大小通常等于`readableBytes()`可读字节数
    -   `readType()`：读取基础数据类型
    -   `getTYPE()`：读取基础数据类型，不同于`readTYPE()`方法，`getTYPE()`不改变`readerIndex`读指针的值
    -   `markReaderIndex()`与`resetReaderIndex()`：前一个方法表示把当前的读指针`readerIndex`保存到`markedReaderIndex`属性中；后一种方法表示把保存在`markedReaderIndex`属性的值恢复到读指针`readerIndex`中，`markedReaderIndex`相当于一个读指针的暂存属性

### 7.5 ByteBuf的引用计数

Netty 的 ByteBuf 的内存回收工作是通过引用计数方式管理的。这样即可以对`Pooled ByteBuf`进行支持，也能够尽快找到那些可以回收的 ByteBuf（非 Pooled）。

>   **说明**
>
>   从 Netty 4 版本开始，新增了 ByteBuf 的池化机制，即创建一个缓冲区对象池，将没有被引用的 ByteBuf 对象放入对象缓存池中，需要时重新从对象缓存池取出，而不需要重新创建。

ByteBuf 引用计数的大致规则如下：

-   在默认情况下，当创建完一个 ByteBuf 时引用计数为 1
-   每次调用`retain()`方法，引用计数 +1
-   每次调用`release()`方法，引用计数 -1
-   如果引用为 0，表示这个 ByteBuf 没有哪个进程引用，它占用的内存需要回收，再次访问这个 ByteBUf 对象将抛出异常

接下来多次调用 ByteBuf 的`retain()`和`release()`方法查看效果

```java
public class ReferenceTest {

  @Test
  public void testRef() {

    ByteBuf buffer = ByteBufAllocator.DEFAULT.buffer();
    Logger.info("after create:" + buffer.refCnt());

    buffer.retain();
    Logger.info("after retain:" + buffer.refCnt());

    buffer.release();
    Logger.info("after release:" + buffer.refCnt());

    buffer.release();
    Logger.info("after release:" + buffer.refCnt());

    // 错误:refCnt: 0,不能再retain
    buffer.retain();
    Logger.info("after retain:" + buffer.refCnt());
  }

}
```

运行结果

```bash
[main|ReferenceTest.testRef] |>  after create:1 
[main|ReferenceTest.testRef] |>  after retain:2 
[main|ReferenceTest.testRef] |>  after release:1 
[main|ReferenceTest.testRef] |>  after release:0 
io.netty.util.IllegalReferenceCountException: refCnt: 0, increment: 1
```

为了确保引用计数不会混乱，在 Netty 的业务处理器开发过程中应该结对使用`retain()`和`release()`：

```java
public void handleMethodA(ByteBuf bytebuf) {
  bytebuf.retain();
  try {
    handleMethodB(byteBuf);
  } finally {
    byteBuf.release();
  }
}
```

在 Netty 流水线上，最后一个 Handler 会调用 ByteBuf 的`release()`方法来释放缓冲区的内存空间。

当 ByteBuf 的引用计数为 0 时，Netty 会进行 ByteBuf 的回收，主要有以下两种场景：

-   如果属于池化的 ByteBuf 内存，则将其放入可以重新分配的 ByteBuf 池，等待下一次分配
-   如果属于未池化的 ByteBuf 缓冲区，需要细分为两种情况：
    -   如果是堆（Heap）结构缓冲，会被 JVM 的垃圾回收机制回收
    -   如果是直接（Direct）内存类型，则会调用本地方法释放外部内存（`unsafe.freeMemory`）

除了 ByteBuf 的成员方法`retain()`和`release()`管理引用计数之外，Netty 还提供了一组用于增加和减少引用计数的通用静态方法：

-   `ReferenceCountUtil.retain(Object)`：增加一次缓冲区引用计数的静态方法，从而防止该缓冲区被释放
-   `ReferenceCountUtil.release(Object)`：减少一次缓冲区引用计数的静态方法，如果引用计数为 0，缓冲区将被释放

### 7.6 ByteBuf分配器

Netty 通过`ByteBufAllocator`分配器来创建缓冲区和分配内存空间。Netty 提供了两种分配器实现：`PoolByteBufAllocator`和`UnpooledByteBufAllocator`。

-   `PoolByteBufAllocator`（池化的 ByteBuf 分配器）将 ByteBuf 实例放入池中，提高了性能，将内存碎片减少到最小；池化分配器采用了 jemalloc 高效内存分配的策略
-   `UnpooledByteBufAllocator`是普通的未池化 ByteBuf 分配器，没有把 ByteBuf 放入池中，每次被调用时，返回一个新的 ByteBuf 实例；使用完之后，通过 Java 的垃圾回收机制回收或者直接释放

在 Netty 中，默认的分配器是`ByteBufAllocator.DEFAULT`。该默认的分配器可以通过系统参数（System Property）选项`io.netty.allocator.type`进行配置，配置时使用字符串值`unpooled`，`pooled`。

>   **提示**
>
>   不同 Netty 版本对于分配器的默认使用策略是不一样的。在 Netty 4.0 版本中，默认的分配器为`UnpooledByteBufAllocator`；在 Netty 4.1 版本中，默认的分配器为`PooledByteBufAllocator`。

初始化代码在 ByteBufUtil 类中的静态代码中：

```java
public final class ByteBufUtil {
  static {
    // Android 系统默认为 unpooled，其他系统默认为 pooled
    // 除非通过系统属性 io.netty.allocator.type 做专门配置
    String allocType = SystemPropertyUtil.get(
      "io.netty.allocator.type",
      PlatformDependent.isAndroid() ? "unpooled" : "pooled"
    );
    ByteBufAllocator alloc;
    if ("unpooled".equals(allocType)) {
      alloc = UnpooledByteBufAllocator.DEFAULT;
      ...
    } else if ("pooled".equals(allocType)) {
      alloc = PooledByteBufAllocator.DEFAULT;
      ...
    } else {
      alloc = PooledByteBufAllocator.DEFAULT;
      ...
    }
    DEFAULT_ALLOCATOR = alloc;
  }
}
```

现在`PooledByteBufAllocator`有了增强的缓冲区泄漏追踪机制，因此可以在 Netty 程序中设置引导类 Bootstrap 装配的时候将`PooledByteBufAllocator`设置为默认的分配器。

```java
ServerBootstrap b = new ServerBootstrap();
b.option(ChannelOption.SO_KEEPALIVE, true);
// 设置父通道的缓冲区分配器
b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
// 设置子通道的缓冲区分配器
b.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
```

使用缓冲区分配器创建 ByteBuf 的方法有多种，下面列出几种主要的：

-   通过默认分配器分配

    -   传入参数，初始化容量为 9，最大容量为 100 的缓冲区

        ```java
        ByteBuf byteBuf = ByteBufAllocator.DEFAULT.buffer(9, 100);
        ```

    -   不传入参数，默认初始容量 256，最到容量为 Integer.MAX_VALUE 的缓冲区

        ```java
        ByteBuf byteBuf = ByteBufAllocator.DEFAULT.buffer();
        ```

-   非池化分配器，分配 Java 的堆（Heap）结构内存缓冲区

    ```java
    ByteBuf byteBuf = UnpooledByteBufAllocator.DEFAULT.heapBuffer();
    ```

-   池化分配器，分配由操作系统管理的直接内存缓冲区

    ```java
    ByteBuf buteBuf = PooledByteBufAllocator.DEFAULT.directBuffer();
    ```

### 7.7 ByteBuf缓冲区

根据内存的管理方式不同，缓冲区分为**堆缓冲区**（Heap ByteBuf）和**直接缓冲区**（Direct ByteBuf），另外还提供了一种组合缓存区。

|       类型        |                             说明                             |                             优点                             |                             不足                             |
| :---------------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|  `Heap ByteBuf`   | 内部数据为一个 Java 数组，存储在 JVM 的对空间中，可以通过`hasArray`方法来判断是不是堆缓冲区 |          未使用池化的情况下，能提供快速的分配和释放          |          写入底层传输通道之前，都会复制到直接缓冲区          |
| `Direct ByteBuf`  |              内部数据存储在操作系统的物理内存中              | 能获取超过 JVM 堆限制大小的内存空间；写入传输通道比堆缓冲区更快 | 释放和分配空间昂贵（使用了操作系统的方法）；在 Java 中读取数据时，需要复制一次到堆上 |
| `CompositeBuffer` |                     多个缓冲区的组合表示                     |                  方便一次操作多个缓冲区实例                  |                                                              |

上面三种缓冲区都可以通过池化、非池化两种分配器来创建和分配内存空间。

>   **Direct Memory**
>
>   -   不属于 Java 对内存，所分配的内存是调用操作系统`malloc()`函数获得，由 Netty 的本地 Native 堆进行管理
>   -   容量可通过`-XX:MaxDirectMemorySize`来指定，如果不指定，则默认与 Java 堆的最大值（`-Xmx`指定）一样（并不是强制要求，有的 JVM 默认 Direct Memory 与`-Xmx`值无直接关系）
>   -   避免了 Java 堆和 Native 堆之间来回复制数据
>   -   由于创建和销毁直接缓冲区的代价较高，因此如果需要频繁创建缓冲区则不适合使用直接缓冲区，尽量在池化分配器中分配和回收
>   -   对直接缓冲区的读写比堆缓冲区快，但是它的创建和销毁比普通堆缓冲区慢
>   -   在 Java 的垃圾回收机制回收 Java 堆时，Netty 框架也会释放不再使用的直接缓冲区，因为它的内存为对外内存，所以清理的工作不会为 JVM 带来压力
>       -   垃圾回收仅在 Java 堆被填满，以至于无法为新的堆分配请求提供服务时发生
>       -   在 Java 应用程序中调用`System.gc()`函数来释放内存

### 7.8 案例：缓冲区的使用

`Heap ByteBuf`和`Direct ByteBuf`两类缓冲区的使用区别：

-   `Heap ByteBuf`通过调用分配器的`buffer()`方法来创建；`Direct ByteBuf`通过调用分配器的`directBuffer()`方法来创建
-   `Heap ByteBuf`缓冲区可以直接通过`array()`方法读取内部数据；`Direct ByteBuf`缓冲区不能读取内部数据
-   可以调用`hasArray()`来判断是否为`Heap ByteBuf`类型的缓冲区；如果`hasArray()`返回值为 true，则表示是堆缓冲，否则为直接内存缓冲区
-   从`Direct ByteBuf`读取缓冲数据进行 Java 程序处理时相对比较麻烦，需要通过`getBytes/readBytes`等方法先将数据复制到 Java 的堆内存，然后进行其他计算

**案例**

```java
public class BufferTypeTest {

  final static Charset UTF8 = Charset.forName("UTF-8");

  // 堆缓存测试
  @Test
  public void testHeapBuffer() {
    ByteBuf heapBuffer = ByteBufAllocator.DEFAULT.heapBuffer();
    heapBuffer.writeBytes("hello world!".getBytes());
    if (heapBuffer.hasArray()) {
      byte[] array = heapBuffer.array();
      int offset = heapBuffer.readerIndex() + heapBuffer.arrayOffset();
      int length = array.length;
      Logger.info(new String(array, offset, length, UTF8));
    }
    heapBuffer.release();
  }

  // 直接缓存测试
  @Test
  public void testDirectBuffer() {
    ByteBuf directBuffer = ByteBufAllocator.DEFAULT.directBuffer();
    directBuffer.writeBytes("hello world!".getBytes());
    if (!directBuffer.hasArray()) {
      int length = directBuffer.readableBytes();
      byte[] array = new byte[length];
      // 把数据读取到 Java 堆内存再进行处理
      directBuffer.getBytes(directBuffer.readerIndex(), array);
      Logger.info(new String(array, UTF8));
    }
    directBuffer.release();
  }

}
```

```bash
[main|BufferTypeTest.testHeapBuffer] |>  hello world! 
[main|BufferTypeTest.testDirectBuffer] |>  hello world! 
```

`Direct ByteBuf`的`hasArray()`会返回 false；但是如果`hasArray()`返回 false，不一定代表缓冲区一定就是`Direct ByteBuf`，也有可能是`CompositeByteBuf`。

Netty 还提供了一个非常方便的获取缓冲区的类：`Unpooled`，可以用来创建和使用非池化的缓冲区，`Unpooled`类可用于 Netty 应用之外的其他程序中独立使用创建 BYteBuf。

|                       方法名称                       |      说明      |
| :--------------------------------------------------: | :------------: |
|                      `buffer()`                      |   返回堆缓冲   |
|            `buffer(int initialCapacity)`             |   返回堆缓冲   |
|    `buffer(int initialCapacity, int maxCapacity)`    |   返回堆缓冲   |
|                   `directBuffer()`                   |  返回直接缓冲  |
|         `directBuffer(int initialCapacity)`          |  返回直接缓冲  |
| `directBuffer(int initialCapacity, int maxCapacity)` |  返回直接缓冲  |
|                 `compositeBuffer()`                  |  返回混合缓冲  |
|                   `copiedBuffer()`                   | 返回复制的缓冲 |

在 Netty 的开发过程中，推荐使用`Context.alloc()`方法获取通道的缓冲区分配器来创建 ByteBuf。

