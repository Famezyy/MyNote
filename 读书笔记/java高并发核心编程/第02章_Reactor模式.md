# 第02章_Reactor模式

## 1.Reactor模式

Reactor（反应器）模式是高性能网络编程在设计和架构层面的基础模式，“Nginx”、“Redis”、“Netty”都是基于 Reactor 模式开发的。

Reactor 模式中有 Reactor 和 Handler 两个重要组件：

-   Reactor：负责查询 IO 事件，当检测到一个 IO 事件时将其发送给相应的 Handler 处理器处理，这里的 IO 事件就是 NIO 中选择器查询出来的通道 IO 事件
-   Handler：与 IO 事件（或选择键）绑定，负责 IO 事件的处理，完成真正的连接建立、通道读取、业务处理、将结果写入通道等

### 1.1 多线程OIO的缺陷

在 Java 的 OIO 编程中，原始的网络服务器一般使用一个 while 循环不断地监听端口是否有新的连接。如果有，就调用一个处理函数来完成处理：

```java
while (true) {
  socket = accept();
  handle(socket);
}
```

这种方式的最大问题是，如果前一个网络连接还没有处理完，那么后面的新连接就无法被服务端接收，导致吞吐量太低。

为了解决这个连接阻塞问题，出现了一个经典的模式：`Connection Per Thread`模式。

```java
public class ConnectionPerThread implements Runnable {

  @Override
  public void run() {
    try {
      ServerSocket serverSocket = new ServerSocket(18888);
      while (!Thread.interrupted()) {
        Socket socket = serverSocket.accept();
				Handler handler = new Handler(socket);
        new Thread(handler).start();
      }
    } catch (IOException e) {
      e.printStackTrace();
    }
  }

  static class Handler implements Runnable {

    final Socket socket;

    public Handler(Socket s) {
      socket = s;
    }

    @Override
    public void run() {
      while (true) {
        try {
          // 读取数据
          byte[] input = new byte[1024];
          socket.getInputStream().read(input);
          // 写入数据
          byte[] output = null;
          socket.getOutputStream().write(output);
        } catch (IOException e) {
          e.printStackTrace();
        }
      }
    }
  }
}
```

以上代码中，对于每一个新的网络连接都分配一个线程来单独处理自己负责的 socket 连接的输入和输出。服务器监听线程也是独立的，任何 socket 连接的输入和输出处理都不会阻塞到后面的 socket 连接的监听和建立。早期版本的 Tomcat 服务器就是这样实现的。

但是对于大量的连接需要耗费大量的线程资源，而且线程的反复创建、销毁、切换也需要代价，因此在高并发的应用场景下，多线程 OIO 的缺陷是致命的。

一个有效的觉觉办法就是使用 Reactor 模式对线程的数量进行控制，做到一个线程处理大量的连接。

### 1.2 单线程Reactor模式

单线程版本的 Reactor 模式就是 Reactor 和 handler 处于一个线程中执行。

这里需要用到`SelectionKey`的几个重要方法：

-   `void attach(Object o)`：将对象附加到选择键，包括 Handler 实例对象
-   `Object attachment()`：从选择键获取附加对象

在 Reactor 模式实现中，一般在选择键注册完成后调用`attach()`将 Handler 实例绑定到选择键；当 IO 事件发生时通过`attachment()`方法取出绑定的 Handler 实例，然后通过该 Handler 实例完成相应的业务处理。

**案例：EchoServer**

读取客户端的输入并回显到客户端。

```java
public class EchoServerReactor implements Runnable {

  private static Logger logger = Logger.getLogger("echoServerReactor");

  Selector selector;
  ServerSocketChannel serverSocket;

  public EchoServerReactor() throws IOException {
    selector = Selector.open();
    serverSocket = ServerSocketChannel.open();
    serverSocket.configureBlocking(false);
    logger.info("服务器开始监听");
    serverSocket.bind(new InetSocketAddress(18888));
    SelectionKey selectionKey = serverSocket.register(selector, 0);
    selectionKey.attach(new AcceptorHandler());

    selectionKey.interestOps(SelectionKey.OP_ACCEPT);
  }

  // 新连接处理器
  class AcceptorHandler implements Runnable {

    @Override
    public void run() {
      try {
        SocketChannel socketChannel = serverSocket.accept();
        logger.info("接收到新的连接：" + socketChannel.getRemoteAddress());
        if (socketChannel != null)
          new EchoHandler(selector, socketChannel);
      } catch (IOException e) {
        e.printStackTrace();
      }
    }
  }

  @Override
  public void run() {
    try {
      while (!Thread.interrupted()) {
        // 限时阻塞查询
        selector.select(1000);

        Set<SelectionKey> selectedKeys = selector.selectedKeys();
        if (selectedKeys == null || selectedKeys.isEmpty())
          continue;

        Iterator<SelectionKey> iterator = selectedKeys.iterator();
        while (iterator.hasNext()) {
          SelectionKey key = iterator.next();
          iterator.remove();
          dispatch(key);
        }
      }
    } catch (IOException e) {
      e.printStackTrace();
    }
  }

  private void dispatch(SelectionKey key) {
    Runnable handler = (Runnable) key.attachment();
    if (handler != null)
      handler.run();
  }

  public static void main(String[] args) throws IOException {
    new Thread(new EchoServerReactor()).start();
  }

}

// 业务逻辑处理器
class EchoHandler implements Runnable {

  private static Logger logger = Logger.getLogger("echoHandler");

  // 用于改变关注的 IO 事件
  final SelectionKey key;
  final SocketChannel socketChannel;
  final ByteBuffer buf = ByteBuffer.allocate(1024);
  static final int RECEIVING = 0, SENDING = 1;
  int state = RECEIVING;

  public EchoHandler(Selector selector, SocketChannel socketChannel) throws IOException {
    this.socketChannel = socketChannel;
    socketChannel.configureBlocking(false);
    key = socketChannel.register(selector, 0);
    key.interestOps(SelectionKey.OP_READ);
    key.attach(this);
  }

  @Override
  public void run() {
    try {
      switch (state) {
          // 如果是接收状态
        case RECEIVING:
          int length = 0;
          while ((length = socketChannel.read(buf)) > 0)
            logger.info(new String(buf.array(), 0, length));
          // 读完后将 buf 切换为读模式
          buf.flip();
          // 读完后注册 write 就绪事件
          key.interestOps(SelectionKey.OP_WRITE);
          state = SENDING;
          break;
        case SENDING:
          socketChannel.write(buf);
          buf.clear();
          key.interestOps(SelectionKey.OP_READ);
          state = RECEIVING;
          break;
        default:
          break;
      }
    } catch (IOException e) {
      e.printStackTrace();
      key.cancel();
      try {
        socketChannel.finishConnect();
      } catch (IOException e1) {
        e1.printStackTrace();
      }
    }
  }
}
```

客户端（用于测试）

```java
public class EchoClient {

  private static Logger logger = Logger.getLogger("echoClient");

  public void start() throws IOException {

    InetSocketAddress address = new InetSocketAddress(18888);

    // 1、获取通道（channel）
    SocketChannel socketChannel = SocketChannel.open(address);
    logger.info("客户端连接成功");
    // 2、切换成非阻塞模式
    socketChannel.configureBlocking(false);
    socketChannel.setOption(StandardSocketOptions.TCP_NODELAY, true);
    // 不断的自旋、等待连接完成，或者做一些其他的事情
    while (!socketChannel.finishConnect()) {

    }
    logger.info("客户端启动成功！");

    // 启动接受线程
    Processor processor = new Processor(socketChannel);
    Commander commander = new Commander(processor);
    new Thread(commander).start();
    new Thread(processor).start();

  }

  static class Commander implements Runnable {
    Processor processor;

    Commander(Processor processor) throws IOException {
      // Reactor初始化
      this.processor = processor;
    }

    @Override
    public void run() {
      while (!Thread.interrupted()) {

        ByteBuffer buffer = processor.getSendBuffer();

        Scanner scanner = new Scanner(System.in);
        while (processor.hasData.get()) {
          logger.info("还有消息没有发送完，请稍等");
          try {
            Thread.sleep(1000);
          } catch (InterruptedException e) {
            e.printStackTrace();
          }

        }
        logger.info("请输入发送内容:");
        if (scanner.hasNextLine()) {

          String next = scanner.nextLine();
          buffer.put((LocalDateTime.now() + " >>" + next).getBytes());

          processor.hasData.set(true);
        }

      }
    }
  }

  static class Processor implements Runnable {
    ByteBuffer sendBuffer = ByteBuffer.allocate(1024);
    ByteBuffer readBuffer = ByteBuffer.allocate(1024);
    protected AtomicBoolean hasData = new AtomicBoolean(false);
    final Selector selector;
    final SocketChannel channel;

    public ByteBuffer getSendBuffer() {
      return sendBuffer;
    }

    Processor(SocketChannel channel) throws IOException {
      // Reactor初始化
      selector = Selector.open();

      this.channel = channel;
      channel.register(selector, SelectionKey.OP_READ | SelectionKey.OP_WRITE);
    }

    @Override
    public void run() {
      try {
        while (!Thread.interrupted()) {
          selector.select();
          Set<SelectionKey> selected = selector.selectedKeys();
          Iterator<SelectionKey> it = selected.iterator();
          while (it.hasNext()) {
            SelectionKey sk = it.next();
            if (sk.isWritable()) {

              if (hasData.get()) {
                SocketChannel socketChannel = (SocketChannel) sk.channel();
                sendBuffer.flip();
                // 操作三：发送数据
                socketChannel.write(sendBuffer);
                sendBuffer.clear();
                hasData.set(false);
              }

            }
            if (sk.isReadable()) {
              // 若选择键的IO事件是“可读”事件,读取数据
              SocketChannel socketChannel = (SocketChannel) sk.channel();

              int length = 0;
              while ((length = socketChannel.read(readBuffer)) > 0) {
                readBuffer.flip();
                logger.info("server echo:" + new String(readBuffer.array(), 0, length));
                readBuffer.clear();
              }

            }
            // 处理结束了, 这里不能关闭select key，需要重复使用
            // selectionKey.cancel();
          }
          selected.clear();
        }
      } catch (IOException ex) {
        ex.printStackTrace();
      }
    }
  }

  public static void main(String[] args) throws IOException {
    new EchoClient().start();
  }
}
```

相对于传统的多线程 OIO，Reactor 模式不再需要启动多条线程，避免了线程上下文切换的开销。但是由于是单线程，当某个 Handler 阻塞时，会导致其他所有的 Handler 都得不到执行，包括负责接听的`AcceptorHandler`和负责输入输出处理的`EchoHandler`。实际生产中基本不会使用单线程 Reactor 模式。

### 1.3 多线程Reactor模式

多线程 Reactor 的演进分为两个方面：

-   升级 Handler，使用线程池
-   升级 Reactor，引入多个 Selector

总体来说，其流程如下：

-   将负责数据传输处理的 IOHandler 处理器的执行放入独立的线程池中，这样就能隔离负责监听的 handler 和负责数据处理的 handler
-   对于多核 CPU 的服务器，可以讲 Reactor 线程拆分为多个 SubReactor 线程，每个线程负责一个选择器的事件轮训

**案例：MultiThreadEchoServerReactor**

-   在 MultiThreadEchoServerReactor 中创建一个 bossReactor 线程负责 bossSelector，绑定 AcceptorHandler 附件，用于监听新连接事件
-   创建两个 workReactor 线程负责两个 workSelector，用于监听读写事件
-   bossSelector 线程
    -   出现新连接时，取出附件并执行 AcceptorHandler 的`run`方法，为每个 channel 创建一个 EchoHandler，并分配一个 workSelector
-   workSelector 线程
    -   出现新的读写事件时，取出附件并执行 EchoHandler 的`run`方法，并利用线程池异步执行读写操作（这样就不会阻塞 workSelector 的轮询）

```java
public class MultiThreadEchoServerReactor {

  private static Logger logger = Logger.getLogger("multiThreadEchoServerReactot");
  private ServerSocketChannel serverSocket;
  // 用于处理连接事件
  private Reactor bossReactor;
  // 用于处理读写事件
  private Selector[] workSelectors = new Selector[2];
  private Reactor[] workReactors = new Reactor[2];
  private AtomicInteger workReactorIndex = new AtomicInteger(0);

  public MultiThreadEchoServerReactor() throws IOException {
    serverSocket = ServerSocketChannel.open();
    serverSocket.configureBlocking(false);
    serverSocket.bind(new InetSocketAddress(19999));

    Selector bossSelector = Selector.open();
    SelectionKey bossKey = serverSocket.register(bossSelector, 0);
    // 将连接处理器注册到 bossSelector 上
    bossKey.attach(new AcceptorHandler());
    bossKey.interestOps(SelectionKey.OP_ACCEPT);
    bossReactor = new Reactor(bossSelector);

    workSelectors[0] = Selector.open();
    workSelectors[1] = Selector.open();

    // 每个 workReactor 线程负责一个 Selector 监听读写事件
    workReactors[0] = new Reactor(workSelectors[0]);
    workReactors[1] = new Reactor(workSelectors[1]);

  }

  private class AcceptorHandler implements Runnable {

    @Override
    public void run() {
      try {
        SocketChannel channel = serverSocket.accept();
        if (channel != null) {
          logger.info("接收到新的连接");
          int index = workReactorIndex.get();
          // 将接下来的读写事件注册到 workdSelector 上
          Selector workSelector = workSelectors[index];
          // 每个 MultiThreadEchoHandler 负责一个 channel 的读和写事件
          new MultiThreadEchoHandler(workSelector, channel);
        }
      } catch (IOException e) {
        e.printStackTrace();
      }
      if (workReactorIndex.incrementAndGet() >= workReactors.length)
        workReactorIndex.set(0);
    }

  }

  private class Reactor implements Runnable {

    private Selector selector;

    public Reactor(Selector selector) {
      this.selector = selector;
    }

    @Override
    public void run() {
      try {
        while (!Thread.interrupted()) {
          selector.select(1000);
          Set<SelectionKey> keys = selector.selectedKeys();
          if (keys == null || keys.isEmpty())
            continue;
          Iterator<SelectionKey> iterator = keys.iterator();
          while (iterator.hasNext()) {
            SelectionKey key = iterator.next();
            dispatch(key);
          }
          keys.clear();
        }
      } catch (IOException e) {
        e.printStackTrace();
      }
    }

    private void dispatch(SelectionKey key) {
      Runnable handler = (Runnable) key.attachment();
      if (handler != null)
        handler.run();
    }
  }

  // 通过 Reactor 开启三个 Selector
  private void startService() {
    new Thred(bossReactor).start();
    for (Reactor reactor : workReactors)
      new Thread(reactor).start();
  }

  public static void main(String[] args) throws IOException {
    new MultiThreadEchoServerReactor().startService();
  }

}

// 用于处理单个 Channel 的读写事件
class MultiThreadEchoHandler implements Runnable {

  private static Logger logger = Logger.getLogger("multiThreadEchoHandler");
  final SocketChannel socketChannel;
  final SelectionKey key;
  final ByteBuffer buf = ByteBuffer.allocate(1024);
  static final int RECEIVING = 0, SENDING = 1;
  int state = RECEIVING;
  static ExecutorService threadPool = Executors.newFixedThreadPool(4);

  public MultiThreadEchoHandler(Selector selector, SocketChannel socketChannel) throws IOException {
    this.socketChannel = socketChannel;
    socketChannel.configureBlocking(false);
    key = socketChannel.register(selector, 0);
    // 先注册读事件
    key.interestOps(SelectionKey.OP_READ);
    key.attach(this);
    // MultiThreadEchoHandler 由 bossSelector 线程执行
    // 这里唤醒的是对应的 workSelector 线程
    selector.wakeup();
    logger.info("新连接创建完成");
  }

  @Override
  public void run() {
    // 异步任务，在独立的线程池中执行
    // 提交数据传输任务到线程池
    // 使得 IO 处理不在 IO 事件轮询线程中执行，在独立的线程池中执行
    threadPool.execute(new AsyncTask());
  }

  // 为了避免同一 channel 的读写操作混乱，这里加上了同步
  // 当监听到读事件，会在读完后改变为监听写事件，然后触发写事件后再变为监听读事件，然后才能监听到下一个读事件
  private synchronized void asynvRun() {
    try {
      if (state == RECEIVING) {
        int length = 0;
        while ((length = socketChannel.read(buf)) > 0)
          logger.info(new String(buf.array(), 0, length));
        buf.flip();
        key.interestOps(SelectionKey.OP_WRITE);
        state = SENDING;
      } else if (state == SENDING) {
        socketChannel.write(buf);
        buf.clear();
        key.interestOps(SelectionKey.OP_READ);
        state = RECEIVING;
      }
    } catch (IOException e) {
      e.printStackTrace();
    }
  }

  class AsyncTask implements Runnable {

    @Override
    public void run() {
      MultiThreadEchoHandler.this.asynvRun();
    }
  }

}
```

### 1.4 Reactor 模式总结

**Reactor 模式和生产者消费者模式对比**

相同点：有点类似于 Reactor 模式，在生产者消费者模式中，一个或多个生产者将事件加入一个队列中，一个或多个消费者主动从这个队列中拉取事件来处理。

不同点：Reactor 模式是基于查询的，没有专门的队列去缓冲存储 IO 事件，查询到 IO 事件之后，反应器会根据不同 IO 选择键将其分发给对应的 handler 来处理。

**Reactor 模式与观察者模式对比**

相同点：在 Reactor 模式中，当查询到 IO 事件后，服务处理程序使用单路/多路分发策略，同步分发这些 IO 事件。观察者模式也被称作发布/订阅模式，他定义了一种依赖关系，让多个观察者同时监听某一个主题。这个主题对象在状态发生变化时会通知所有观察者执行相应的处理。

不同点：在 Reactor 模式中，每个 IO 事件被查询后，反应器会将事件分发给所绑定的 Handler，也就是一个事件只能被一个 Handler 处理；而在观察者模式中，同一时刻、同一主题可以被订阅过的多个观察者处理。

**Reactor 模式的优缺点**

优点

-   响应快，单线程即可处理多个连接以及相应的读写操作，避免了线程切换的开销
-   编程相对简单，避免了复杂的多线程同步
-   可扩展，可方便地通过增加反应器线程的个数来充分利用 CPU 资源

缺点

-   有一定的复杂性，不易与调试
-   依赖于操作系统底层的 IO 多路复用系统调用的支持，如 Linux 的 epoll

