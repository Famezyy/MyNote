# 第1章_IO简介与NIO详解

## 1.IO读写的基本原理

操作系统将内存（虚拟内存）划分为**内核空间**和**用户空间**。用户进程进行系统调用时必须切换为内核态。

用户程序进行 IO 的读写依赖于底层的 IO 读写，会用到`read`和`write`两大系统调用。`read`会把数据从内核缓冲区复制到应用程序的进程缓冲区，`write`会把数据从应用程序的进程缓冲区复制到操作系统的内核缓冲区。它们不负责数据在内核缓冲区和物理设备之间的交换。这个底层的读写交换操作是由操作系统内核（Kernel）完成。

## 2.四种主要的IO模型

阻塞 IO 指的是需要**内核 IO 操作彻底完成后**才返回到用户空间执行用户程序的操作指令。阻塞指的是用户程序的执行状态。

非阻塞 IO 指的是用户空间的程序不需要等待内核 IO 操作彻底完成，可以立即返回用户空间执行后续的指令。与此同时，内核会立即返回给用户一个 IO 状态值。

同步 IO 指用户空间是主动发起 IO 请求的一方，系统内核是被动接受方。

异步 IO 是指系统内核是主动发起 IO 请求的一方，用户空间是被动接受方。

内核 IO 操作：等待内核缓冲区数据 - 复制到用户缓冲区 - 复制完成

### 2.1 同步阻塞IO

用户进程主动发起，需要等待内核 IO 操作彻底完成后才返回到用户空间的 IO 操作。优点是阻塞期间用户线程挂起，不会占用 CPU 资源，缺点是每个 IO 操作都需要创建一个线程，否则无法执行后续操作。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230108030608945-c56167152022504beeaaa241dd5c8ae8-e27dc2.png" alt="image-20230108030608945" style="zoom:67%;" />

### 2.2 同步非阻塞IO

用户进程主动发起，不需要等待内核 IO 操作彻底完成就能立即返回用户空间的 IO 操作。优点是实时性较好，缺点是需要不断轮询内核，占用大量的 CPU 时间。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230108030634510-be4e92d736b0ba3b9a68981668f2e1a3-eaa6b2.png" alt="image-20230108030634510" style="zoom:67%;" />

> **注意**
>
> 当内核数据没有准备好时，用户线程的 IO 请求会立即返回。但是当内核数据到达后，用户线程发起系统调用==依然会阻塞==，内核会将内核缓冲区复制到用户缓冲区，然后内核返回结果。

### 2.3 IO多路复用

通过 select/epoll 系统调用，一个用户进程（或线程）可以监视多个文件描述符，一旦某个描述符就绪（可读/可写），内核就能够将文件描述符的就绪状态返回给用户进程（或线程），用户可以根据文件描述符的就绪状态进行相应的 IO 系统调用。缺点是读写操作是由系统调用本身负责，依然是阻塞的。

> **说明**
>
> 在用户线程进行 IO 就绪事件的轮询时，需要调用选择器的 select 查询方法，发起查询的用户线程是阻塞的，如果使用了非阻塞的重载方法则不会阻塞。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230108030700606-f9a17afe3ac5afca723b3f54f85a99c8-9d4631.png" alt="image-20230108030700606" style="zoom:67%;" />

**流程**

1. 选择器注册，首先将需要 read 操作的目标文件描述符（socket 连接）注册到 Linux 的 select/epoll 选择器（对应于 Java 的 Selector 类）
2. 就绪状态的轮询，轮询所有注册过的文件描述符的 IO 就绪状态。如果没有就绪设备，则挂起当前线程，直到设备就绪或主动超时，被唤醒后会再次轮询。通过查询（select）的系统调用，内核会返回一个就绪的文件描述符列表。当内核缓冲区中有对应的文件描述符的数据后，内核就将该文件描述符加入就绪列表，返回就绪事件
3. 用户线程获取就绪状态列表后，根据其中的 socket 连接发起 read 系统调用，用户线程阻塞，内核会将内核缓冲区的数据复制到用户缓冲区
4. 复制完成后，内核返回结果，用户线程解除阻塞状态读取到数据

> **解除 Linux 操作系统中文件描述符的限制**
>
> Linux 系统中文件可分为普通文件、目录文件、链接文件和设备文件。文件描述符是内核为了高效管理已被打开的文件锁创建的索引。Linux 系统默认一个进程最多可以处理 1024 个文件描述符，通过`ulimit -n`查看，通过`ulimit -n 1000000`更改，但是仅对当前用户环境有效。
>
> 可以编辑`/etc/rc.local`开机启动文件，添加`ulimit -SHn 1000000`。`-S`表示软性极限值，`-H`表示硬性极限值。硬性极限值是实际的最大限制，软性极限值是系统发出警告的极限值。
>
> 要彻底解除 Linux 系统的最大文件打开数量的限制，可以通过编辑极限配置文件`/etc/security/limits.conf`，加入以下内容：
>
> ```bash
> # 软性极限
> soft nofile 1000000
> # 硬性极限
> hard nofile 1000000
> ```

### 2.4 异步IO

用户线程通过系统调用向内核注册某个 IO 操作，内核在整个 IO 操作（包括数据准备、数据复制）完成后通知用户程序。理论上异步 IO 才是真正的异步输入输出，吞吐量高于 IO 多路复用模型。

> Windows 系统下通过 IOCP 实现了真正的异步 IO，在 Linux 系统下，异步 IO 模型在 2.6 版本才引入，JDK 对它的支持并不完善。因此目前高并发网络应用程序的开发大多采用 IO 多路复用模型。

## 3.Java NIO

OIO 是面向流的，NIO 是面向缓冲区的，可以随意读取 Buffer 中任意位置的数据。它使用了多路复用技术实现了非阻塞，即调用`read`方法会直接返回，底层是基于选择器的系统调用。

三个核心组件：

1. **Channel（通道）**

   在 OIO 中一个网络连接会关联两个流：输入流和输出流，在 NIO 中，一个网络连接使用一个通道表示，类似于 OIO 中两个流的结合体，可以从通道读取和写入数据。

2. **Buffer（缓冲区）**

   读取时将数据从通道中读取到缓冲区中，写入时将数据从缓冲区中写入到通道中。

3. **Selector（选择器）**

   用来监视文件描述符的就绪事件，通过选择器，一个线程可以查询多个通道的 IO 事件的就绪状态。

### 3.1 Buffer

NIO 的 Buffer 本质上是一个内存块（数组），但是具体的内存缓冲区（数组）是定义在子类中的。既可以写入数据也可以读取数据，非线程安全。

Java 中的 Buffer 类位于 java.nio 包下，是一个抽象类，有 8 个子类：`ButeBuffer`、`CharBuffer`、`DoubleBuffer`、`FloatBuffer`、`IntBuffer`、`LongBuffer`、`ShortBuffer`、`MappedByteBuffer`。

#### 1.三个重要属性

1. **capacity**

   表示能写入的数据对象的最大限制数量，一旦写入对象数量超过了就不能再向缓冲区写入。一旦初始化就不能改变，因为数组内存一旦分配大小就不能改变。

2. **position**

   表示当前可写入或读取的位置。读写模式通过`flip()`方法进行切换。

3. **limit**

   表示可以写入或读取的数据最大上限。

**例子**

新创建的缓冲区处于写模式，position 为 0，limit 为 capacity。每写入一个数据，position 向后移动一个位置，假定写入了 5 个数据，此时 position 为 5。最后使用 flip 方法切换到读模式，limit 重置为 position 的值 5，position 重置为 0。

此外，Buffer 还有一个**mark**属性，可以将当前的 position 值临时存入 mark 中 - 调用`mark()`，需要的时候从 mark 中取出暂存的标记值恢复到 position 属性中 - 调用`reset()`。

#### 2.重要方法

##### 2.1 `allocate()`

通过调用子类的`allocate()`方法来创建 Buffer 实例，并且分配内存空间，刚创建好的缓冲区处于写模式。

例如：

```java
IntBuffer intBuffer = IntBuffer.allocate(20);
intBuffer.position(); // 0
intBuffer.capacity(); // 20
intBuffer.limit(); // 20
```

##### 2.2 `put()`

使用`allocate()`创建了 Buffer 对象后，就可以使用`put()`方法将数据写入缓冲区。

书接上例：

```java
for (int i = 0; i < 5; i++) {
  intBuffer.put(i);
}
intBuffer.position(); // 5
intBuffer.capacity(); // 20
intBuffer.limit(); // 20
```

##### 2.3 `flip()`

向缓冲区写入数据后，如果想要读取数据必须使用`flip()`切换到“读模式”。

书接上例：

```java
intBuffer.flip();
intBuffer.position(); // 0
intBuffer.capacity(); // 20
intBuffer.limit(); // 5
```

`flip()`源码可简化为：

```java
public final Buffer flip() {
  limit = position; // 将可读上限 limit 设置为写模式下的 position
  position = 0; // 将读起始位置设置为 0
  mark = UNSET_MARK; // 清楚写模式下的临时位置标记，避免混乱
  return this；
}
```

> 再次切换回写模式时，可以调用`Buffer.clear()`清空，或者`Buffer.compact()`压缩方法。

##### 2.4 `get()`

切换为“读模式”后就可以调用`get()`每次从 position 处读取一个数据，并对缓冲区属性进行调整。

书接上例：

```java
for (int i = 0; i < 2; i++) {
  int j = intBuffer.get();
}
intBuffer.position(); // 2
intBuffer.capacity(); // 20
intBuffer.limit(); // 5

for (int i = 0; i < 3; i++) {
  int j = intBuffer.get();
}
intBuffer.position(); // 5
intBuffer.capacity(); // 20
intBuffer.limit(); // 5
```

当 position 与 limit 相等时即表示数据读取完成，position 指向了一个没有数据的元素位置，此时再读就会抛出`BufferUnderFlowException`。此时除非清空或者压缩缓冲区，将其切换为写模式，否则不能进行写入。

> 需要重复读时，即可以使用倒带方法`rewind()`，也可以通过`mark()`和`reset()`两个方法组合完成。

##### 2.5 `rewind()`

已经读完的数据需要再读一次，可以调用`rewind()`。

书接上例：

```java
intBuffer.rewind();
intBuffer.position(); // 0
intBuffer.capacity(); // 20
intBuffer.limit(); // 5
```

`rewind()`源码可简化为：

```java
public final Buffer rewind() {
  position = 0;
  mark = -1;
  return this;
}
```

> **与`flip()`区别**
>
> `flip()`会重设 limit，`rewind()`不会重设 limit。

`rewind()`倒带之后，就可以重新读取数据：

```java
for (int i = 0; i < 5; i++) {
  if (i == 2)
    intBuffer.mark();
  int j = inBuffer.get();
}
intBuffer.position(); // 5
intBuffer.capacity(); // 20
intBuffer.limit(); // 5
```

##### 2.6 `mark()`和`reset()`

`Buffer.mark()`方法将当前 position 值保存到 mark 成员属性中，然后调用`Buffer.reset()`方法可以讲 mark 的值恢复到 position 中。

书接上例，在`i=2`时调用了`mark()`方法将当前 position 的值存储到 mark 中，此时 mark 值为 2。

```java
intBuffer.reset();
intBuffer.position(); // 2
intBuffer.capacity(); // 20
intBuffer.limit(); // 5

for (int i = 2; i < 5; i++) {
  intBuffer.get(); // 依次输出 2，3，4
}
```

##### 2.7 `clear()`

在读模式下，调用`clear()`方法可以切换到写模式。此方法的作用为：

- 将 position 清零
- limit 设置为 capacity 最大容量值

书接上例：

```java
intBuffer.clear();
intBuffer.position(); // 0
intBuffer.capacity(); // 20
intBuffer.limit(); // 20
```

##### 2.8 基本步骤

1. 使用创建子类实例对象的`allocate()`方法创建一个 Buffer 对象
2. 调用`put()`将数据写入缓冲区
3. 调用`flip()`切换为“读模式”
4. 调用`get()`从缓冲区读取数据
5. 读去完成后，调用`clear()`或者`compact()`切换为写模式，可以继续写入

### 3.2 Channel

不同的网络传输协议类型，在 Java 中都有不同的 NIO Channel 实现，比较重要的有：

- FileChannel：文件通道，用于文件的数据读写
- SocketChannel：套接字通道，用于套接字 TCP 连接的数据读写
- ServerSocketChannel：服务器套接字通道，用于监听 TCP 连接请求，为每个监听到的请求创建一个 SocketChannel 通道
- DatagramChannel：数据报通道，用于 UDP 的数据读写

#### 1.FileChannel

通过 FileChannel 可以从文件中读取数据和向文件中写入数据，仅支持阻塞模式。

##### 1.1 获取FileChannel

可以通过文件的输入流、输出流来获取：

```java
FileInputStream fis = new FileInputStream(srcFile);
FileChannel inChannel = fis.getChannel();

FileOutStream fos = new FileOutputStream(destFile);
FileChannel outChannel = fos.getChannel();
```

也可以通过`RandomAccessFile`（文件随机访问）类来获取 FileChannel：

```java
RandomAccessFile rFile = new RandomAccessFile("filename.txt", "rw");
FileChannel channel = rFile.getChannel();
```

##### 1.2 读取FileChannel

一般情况下，从通道读取数据都会调用通道的`int read(ByteBuffer buf)`，把从通道读取的数据写入 ByteBuffer，并返回读取数据的数据量。

```java
RandomAccessFile rFile = new RandomAccessFile(filename, "rw");
FileChannel channel = rFile.getChannel();
ByteBuffer buf = ByteBuffer.allocate(CAPACITY);
int length = -1;
while ((length = channel.read(buf)) != -1)
  doSomething()
```

##### 1.3 写入FileChannel

一般情况下，向通道中写入数据都会调用通道的`int write(ByteBuffer buf)`，把从 ByteBuffer 缓冲区中读到的数据写入通道，返回写入成功的字节数。

```java
// 如果 buf 处于写模式，需要切换为读模式
buf.flip();
int outlength = 0;
while ((outlength = outchannel.write(buf)) != 0)
  doSomething()
```

##### 1.4 关闭通道

使用完通道后需要调`close()`关闭。

```java
channel.close();
```

##### 1.5 强制刷新到磁盘

将缓冲区写入通道时，由于性能原因操作系统不会每次都实时de写入数据到磁盘，要保证缓冲区数据能写入磁盘，可以在写入后调用`force()`。

```java
channel.force();
```

##### 1.6 案例：文件复制

```java
public static void main(String[] args) throws Exception {
  // 文件路径
  String srcFilePath = ThreadTest.class.getResource("test.txt").getPath();
  String descFilePath = ThreadTest.class.getResource("test.txt").getFile().replace("test.txt", "test2.txt");

  // 文件对象
  File srcFile = new File(srcFilePath);
  File descFile = new File(descFilePath);

  if (!descFile.exists())
    descFile.createNewFile();

  try(
    FileInputStream fis = new FileInputStream(srcFile);
    FileOutputStream fos = new FileOutputStream(descFile);
    FileChannel inChannel = fis.getChannel();
    FileChannel outChannel = fos.getChannel();
  ) {
    // 新建 buf，处于写模式
    ByteBuffer buf = ByteBuffer.allocate(1024);
    while (inChannel.read(buf) != -1) {
      // 从写模式切换为读模式
      buf.flip();
      int outLength = 0;
      while ((outLength = outChannel.write(buf)) != 0) {
        logger.info("写入的字节数：" + outLength);
      }
      // 从读模式切换为写模式，并重置
      buf.clear();
    }
    // 强制刷新
    outChannel.force(true);
  }
}
```

以上代码用于演示文件通道以及字节缓冲区的作用，但效率不是最高的，更高效的文件复制可以调用文件通道的`transferFrom()`。

```java
try(
  FileInputStream fis = new FileInputStream(srcFile);
  FileOutputStream fos = new FileOutputStream(descFile);
  FileChannel inChannel = fis.getChannel();
  FileChannel outChannel = fos.getChannel();
) {
  outChannel.transferFrom(inChannel, inChannel.position(), inChannel.size());
  outChannel.force(true);
}
```

#### 2.SocketChannel

在 NIO 中涉及到网络连接的通道有两个：负责连接的数据传输通道`SocketChannel`和负责监听连接的`ServerSocketChannel`。`SocketChannel`对应于 BIO 的`Socket`，`ServerSocketChannel`对应于 BIO 的`ServerSocket`。

`ServerSocketChannel`仅用于服务端，而`SocketChannel`同时应用于服务端和客户端。它们都支持**阻塞**和**非阻塞**两种模式，可通过`configureBlocking()`设置：

```java
socketChannel.configureBlocking(false);
```

>   **说明**
>
>   NIO 套接字通道主要用于非阻塞传输场景，因此基本上都需要调用通道的`configureBlocking(false)`将通道从阻塞模式切换到非阻塞模式。

在阻塞模式下，SocketChannel 的连接、读、写操作都是同步阻塞的，在效率上与 Java BIO 面向流的阻塞式读写操作相同。在非阻塞模式下，通道的操作是异步、高效的，下面将重点关注非阻塞模式下的操作。

##### 2.1 获取SocketChannel传输通道

```java
// 获得一个套接字传输通道
SocketChannel socketChannel = SocketChannel.open();
// 设置为非阻塞模式
socketChannel.configureBlocking(false);
// 对服务器的 IP 和端口发起连接
socketChannel.connect(new InetSocketAddress("127.0.0.1", 80));
```

在非阻塞模式下，与服务器的连接可能还没有真正建立`connect()`方法就返回了，因此需要自旋检查是否建立了连接：

```java
while(!socketChannel.finishConnect()) {
  // 自旋等待，可打印日志
}
```

##### 2.2 读取SocketChannel传输通道

当 SocketChannel 传输通道**可读**时，可以调用`read()`将数据读入缓冲区`ByteBuffer`：

```java
ButeBuffer buf = ButeBuffer.allocate(1024);
int bytesRead = socketChannel.read(buf);
```

`read()`如果返回 -1 则表示读取到对方的输出结束标志，即对方准备关闭连接。这里的难点是在非阻塞模式下如何知道通道是可读的，这需要用到 NIO 的 Selector 通道选择器。

##### 2.3 写入SocketChannel传输通道

```java
// 新创建的 buffer 是写模式，需要切换到读模式
buffer.flip();
socketChannel.write(buffer)；
```

##### 2.4 关闭SocketChannel传输通道

在关闭 SocketChannel 通道前，如果传输通道是用来写入数据的，则建议调用一次`shutdownOutput()`向对方发送一个输出的结束标志（-1），然后调用`close()`方法关闭套接字连接。

```java
socketChannel.shutdownOutput();
socketChannel.close();
```

##### 2.5 案例：发送文件

使用 FileChannel 读取本地文件内容，通过 SocketChannel 发送到服务器。

文件发送过程是：发送文件名称长度 - 发送文件名称 - 发送文件内容长度 - 发送文件内容

```java
public static void sendFile() {

  String srcFilePath = NioSendClient.class.getResource("IMG_5640.JPG").getPath();
  String descFileName = "copy.JPG";

  File srcFile = new File(srcFilePath);
  try (FileChannel srcFileChannel = new FileInputStream(srcFile).getChannel();
       SocketChannel socketChannel = SocketChannel.open();) {
    socketChannel.configureBlocking(false);
    socketChannel.connect(new InetSocketAddress("127.0.0.1", 18899));
    while (!socketChannel.finishConnect()) {
    }

    sendFileNameAndLength(descFileName, socketChannel);
    sendFileContentAndLength(srcFile, srcFileChannel, socketChannel);

    socketChannel.shutdownOutput();

  } catch (IOException e) {
    e.printStackTrace();
  }
}

private static void sendFileContentAndLength(File srcFile, FileChannel srcFileChannel, SocketChannel socketChannel)
  throws IOException {
  ByteBuffer fileContentlengthBuf = ByteBuffer.allocate(8);
  // 设置文件长度，8 个字节
  fileContentlengthBuf.putLong(srcFile.length());
  fileContentlengthBuf.flip();
  socketChannel.write(fileContentlengthBuf);
  fileContentlengthBuf.clear();

  // 发送文件内容
  ByteBuffer buf = ByteBuffer.allocate(512);
  while (srcFileChannel.read(buf) > 0) {
    buf.flip();
    socketChannel.write(buf);
    buf.clear();
  }
  logger.info("sent file successfully");
}

private static void sendFileNameAndLength(String descFileName, SocketChannel socketChannel) throws IOException {
  Charset charset = Charset.forName("UTF-8");
  ByteBuffer fileNameLengthBuffer = ByteBuffer.allocate(4);

  // 设置文件名称长度，4 个字节
  ByteBuffer fileNameByteBuffer = charset.encode(descFileName);
  int fileNameLength = fileNameByteBuffer.capacity();
  fileNameLengthBuffer.putInt(fileNameLength);
  fileNameLengthBuffer.flip();
  socketChannel.write(fileNameLengthBuffer);
  fileNameLengthBuffer.clear();

  // 设置文件名称
  socketChannel.write(fileNameByteBuffer);
  fileNameByteBuffer.clear();
  logger.info("sent file name：" + descFileName);
}
```

#### 3.DatagramChannel

在 Java NIO 中，使用 DatagramChannel 来处理 UDP 的数据传输，UDP 不是面向连接的协议，只需知道服务器的 IP 和端口就可以发送数据。

##### 3.1 获取DatagramChannel

```java
// 获取 DatagramChannel
DatagramChannel channel = DatagramChannel.open();
// 设置非阻塞模式
datagramChannel.configureBlocing(false);
```

如果需要接受数据，还需要调用`bind()`方法绑定一个数据报的监听端口：

```java
channel.socket().bind(new InetSocketAddress(18080));
```

##### 3.2 读取数据

与`SocketChannel`使用`read()`不同，这里调用`receive(ByteBuffer buf)`方法将 DatagramChannel 通道中的数据读入缓冲区：

```java
ByteBuffer buf = ByteBuffer.allocate(1024);
SocketAddress clientAddr = datagramChannel.receive(buf);
```

`receive()`方法在将数据写入缓冲区的同时，会返回发送端的连接地址（包括 IP 和端口）。和 SocketChannel 一样，DatagramChannel 也需要通过 Selector 获知何时通道是可读的。

##### 3.3 写入DatagramChannel

和`SocketChannel`使用`write()`不同，这里调用`send()`方法：

```java
buffer.flip();
dataChannel.send(buffer, new InetSocketAddress("127.0.0.1", 18899));
buffer.clear();
```

##### 3.4 关闭DatagramChannel

```java
dataChannel.close();
```

##### 3.5 案例：发送数据

客户端：获取用户的输入，通过 DatagramChannel 发送到服务器。

```java
private static void sendMessage() {
  try (DatagramChannel dChannel = DatagramChannel.open()) {
    dChannel.configureBlocking(false);
    ByteBuffer buf = ByteBuffer.allocate(1024);

    Scanner sc = new Scanner(System.in);
    while (sc.hasNextLine()) {
      String s = sc.nextLine();
      buf.put(s.getBytes());
      buf.flip();
      dChannel.send(buf, new InetSocketAddress("127.0.0.1", 18899));
      buf.clear();
    }
  } catch (IOException e) {
    e.printStackTrace();
  }
}
```

服务端：通过 DatagramChannel 绑定一个服务器地址，接受客户端数据。

这里用到了后面会介绍的 Selector。

```java
private static void receiveMessage() {
  try (DatagramChannel datagramChannel = DatagramChannel.open(); Selector selector = Selector.open()) {
    datagramChannel.configureBlocking(false);
    datagramChannel.bind(new InetSocketAddress("127.0.0.1", 18899));
    datagramChannel.register(selector, SelectionKey.OP_READ);

    while (selector.select() > 0) {
      Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
      ByteBuffer buf = ByteBuffer.allocate(1024);
      while (iterator.hasNext()) {
        SelectionKey selectionKey = iterator.next();
        if (selectionKey.isReadable()) {
          datagramChannel.receive(buf);
          buf.flip();
          logger.info(new String(buf.array(), 0, buf.limit()));
          buf.clear();
        }
      }
      iterator.remove();
    }
  } catch (IOException e) {
    e.printStackTrace();
  }
}
```

### 3.3 Selector

选择器用于完成 IO 的多路复用，主要工作是负责通道的注册、监听、事件查询。一个通道代表一条连接通路，通过选择器可以同时监控多个通道的 IO 状况。

选择器提供了独特的 API 方法，能够**选出（select）**所监控的通道发生了哪些 IO 事件。一般一个单线程处理一个选择器，一个选择器可以监控很多通道。在极端情况下（数万个连接），只用一个线程就可以处理所有的通道，大大减少了线程间上下文切换的开销。

通道和选择器间的关联通过**注册（register）**完成。调用通道的`Channel.register(Selector sel, int ops)`可以讲通道实例注册到一个选择器中，第一个参数指定通道注册到的选择器实例，第二个参数指定选择器要监控的 IO 事件类型。

可供选择器监控的通道 IO 事件类型包括以下四种：

-   可读：`SelectionKey.OP_READ`
-   可写：`SelectionKey.OP_WRITE`
-   连接：`SelectionKey.OP_CONNECT`
-   接收：`SelectionKey.OP_ACCEPT`

如果选择器要监控多种事件，可以用"按位或"运算符来实现。例如，同时监控可读和可写 IO 事件：

```java
int key = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
```

这里的 IO 事件不是对通道的 IO 操作，而是通道处于某个 IO 操作的就需状态，表示通道具备执行某个 IO 操作的条件。例如：

-   一个 SocketChannel 通道有数据可读，就会发生“读就绪”事件
-   一个 SocketChannel 通道等待数据写入，就会发生“写就绪”事件

-   某个 SocketChannel 传输通道如果完成了和对端的三次握手，就会发生“连接就绪”事件
-   某个 ServerSocketChannel 服务器连接监听通道，在监听到一个新连接到来时就会发生“接收就绪“事件

#### 1.SelectableChannel

只就继承了抽象类`SelectableChannel`的通道才可以被选择器监控或选择，如`FileChannel`没有继承`SelectableChannel`，所以就不能被选择器复用。而 NIO 中所有网络连接 socket 通道都继承了`SelectableChannel`，因此都是可选择的。

#### 2.SelectionKey

一但在通道中发生了某些 IO 事件（就需状态达成），并且是在选择器中注册过的 IO 事件，就会被选择器选中，并放入`SelectionKey	`的集合中。通过`SelectionKey`，不仅可以获得通道的 IO 事件类型，还可以获得发生 IO 事件所在的通道，还可以获得选择器实例。

#### 3.选择器使用流程

##### 3.1 获取选择器实例

通过调用静态工厂方法`open()`来获取：

```java
Selector selector = Selector.open();
```

>   **扩展**
>
>   `Selector`的类方法`Open()`的内部是向选择器 SPI 发出请求，通过默认的`SelectorProvider`对象获取一个新的选择器实例。Java 中的 SPI（Service Provider Interface，服务提供者接口）是一种可以扩展的服务提供的发现机制。Java 通过 SPI 的方式提供选择器的默认实现版本。其他的服务提供者也可以通过 SPI 的方式提供定制化版本的选择器的动态替换或扩展。

##### 3.2 将通道注册到选择器实例

```java
ServerSocketChannel serverSocketChannel = ServerSockerChannel.open();
serverSocketChannel.configureBlocking(false);
serverSocketChannel.bind(new InetSocketAddress(18899));
serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
```

这里需要注意：注册到选择器的通道必须处于非阻塞模式下，否则将抛出`IllegalBlockingModeException`异常。其次，一个通道不一定支持所有的四种 IO 事件。例如，服务器监听通道`ServerSocketChannel`仅支持`Accept`（接受到新连接）IO  事件，而传输通道 SocketChannel 不支持`Accept`类型的 IO 事件。

可以在注册之前通过通道的`validops()`方法来获取该通道支持的所有 IO 事件集合。

##### 3.3 选出感兴趣的IO就绪事件

通过`Selector`的`select()`方法选出已经注册的、已经就绪的 IO 事件，并保存到`SelectionKey`集合中。`SelectionKey`集合保存在选择器实例内部，调用`selectedKeys()`方法可以获取选择键集合。

接下来，迭代集合的每一个选择键，根据具体 IO 事件类型执行对应的业务操作。

大致流程如下：

```java
while (selector.select() > 0) {
  Iterator keyIterator = selector.selectedKeys().iterator();
  while (keyIterator.hasNext()) {
    SelectionKey key = keyIterator.next();
    if (key.isAcceptable()) {
      // IO 事件：ServerSocketChannel 服务器监听通道有新连接
    } else if (key.isConnectable()) {
      // IO 事件：传输通道连接成功
    } else if (key.isReadable()) {
      // IO 事件：传输通道可读
    } else if (key.isWritable()) {
      // IO 事件：传输通道可写
    }
    keyIterator.remove();
  }
}
```

处理完成后需要将选择键从`SelectionKey`集合中移除，防止下一次循环时被重复处理。`SelectionKey`集合不能添加元素，如果试图向`SelectionKey`中添加元素，则将抛出`java.lang.UnsupportedOperationException`异常。

`select()`方法有多个重载的实现版本：

-   `select()`：阻塞调用，直到zhishaoyouyige通道发生了注册的 IO 事件
-   `select(long timeout)`：和`select()`一样，但最长阻塞事件为`timeout`指定的毫秒数
-   `selectNow()`：非阻塞，不管有没有 IO 事件都会立刻返回

`select()`的返回值是`int`类型，表示从上一次 select 到这一次 select 之间发生的注册过的 IO 事件数。

##### 3.4 案例：Discard服务器

功能为：仅读取客户端通道的输入数据，读取完成后直接关闭客户端通道，并且直接抛弃到读取到的数据。

```java
private static void startServer() {
  try (
    // 1.获取选择器
    Selector selector = Selector.open();
    // 2.获取通道
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open()) {
    // 3.设置非阻塞
    serverSocketChannel.configureBlocking(false);
    // 4.绑定连接
    serverSocketChannel.bind(new InetSocketAddress(18899));
    // 5.将通道注册到“接受新连接” IO 事件注册到选择器上
    serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
    // 6.阻塞等待感兴趣的 IO 就绪事件
    while (selector.select() > 0) {
      // 7.获取选择键集合
      Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
      while (iterator.hasNext()) {
        // 8.获取单个选择键
        SelectionKey selectionKey = iterator.next();
        // 9.判断 key 的具体事件类型
        if (selectionKey.isAcceptable()) {
          // 10.若选择键的 IO 事件是“连接就绪”，就获取客户端连接
          SocketChannel socketChannel = serverSocketChannel.accept();
          if (socketChannel == null) continue;
          // 11.将新连接切换为非阻塞模式
          socketChannel.configureBlocking(false);
          // 12.将新连接的通道的可读事件注册到选择器上
          socketChannel.register(selector, SelectionKey.OP_READ);
        } else if (selectionKey.isReadable()) {
          // 13.若选择键的 IO 事件是“可读”，则读取数据
          SocketChannel sockerChannel = (SocketChannel) selectionKey.channel();
          // 14.读取数据，然后丢弃
          ByteBuffer buf = ByteBuffer.allocate(1024);
          while (sockerChannel.read(buf) > 0) {
            buf.flip();
            logger.info(new String(buf.array(), 0, buf.limit()));
            buf.clear();
          }
          socketChannel.close();
        }
        // 15.移除选择键
        iterator.remove();
      }
    }
  } catch (IOException e) {
    // TODO Auto-generated catch block
    e.printStackTrace();
  }
}
```

在以上程序中涉及两次选择器注册：一次是注册 serverChannel；另一次是注册接受到的 sockerChannel 客户端传输通道。

客户端代码如下：

```java
private static void startClient() {
  try (SocketChannel socketChannel = SocketChannel.open()) {
    socketChannel.configureBlocking(false);
    socketChannel.connect(new InetSocketAddress(18899));
    while (!socketChannel.finishConnect()) {
    }
    ByteBuffer buf = ByteBuffer.allocate(1024);
    buf.put("hello world".getBytes());
    buf.flip();
    socketChannel.write(buf);
    socketChannel.shutdownOutput();
    buf.clear();
  } catch (IOException e) {
    e.printStackTrace();
  }
}
```

##### 3.5 案例：服务端接收文件

本示例演示文件的接受，和前面的<a href="#2.5 案例：发送文件">发送文件</a>的 SocketCHannel 客户端程序相互配合。

```java
private static ConcurrentHashMap<SocketAddress, Client> CLIENT_MAP = new ConcurrentHashMap<>();
private static String FILE_PATH = NioReceiveServer.class.getResource("/").getPath();
private static Charset UTF8 = Charset.forName("UTF-8");

private static class Client {
  ByteBuffer buf;
  FileOutputStream fos;
  FileChannel fileChannel;
  String fileName;
  int fileNameLength;
  long fileContentLength;
  int step;

  public FileChannel getChannel(File file) {
    try {
      this.fos = new FileOutputStream(file);
      this.fileChannel = fos.getChannel();
    } catch (FileNotFoundException e) {
      e.printStackTrace();
    }
    return this.fileChannel;
  }

  public void shutdown() {
    try {
      buf.clear();
      fos.close();
      fileChannel.close();
    } catch (IOException e) {
      // TODO Auto-generated catch block
      e.printStackTrace();
    }
  }
}

private static void startServer() {
  // configure serverSockerChannel
  try (ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
       Selector selector = Selector.open()) {
    serverSocketChannel.configureBlocking(false);
    serverSocketChannel.bind(new InetSocketAddress(18899));
    serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
    logger.info("wait for connect");
    while (selector.select() > 0) {
      Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
      while (iterator.hasNext()) {
        SelectionKey selectionKey = iterator.next();
        // if access IO
        if (selectionKey.isAcceptable()) {
          ServerSocketChannel server = (ServerSocketChannel) selectionKey.channel();
          SocketChannel socketChannel = server.accept();
          if (socketChannel == null)
            continue;
          socketChannel.configureBlocking(false);
          socketChannel.register(selector, SelectionKey.OP_READ);
          // 发生半包时，会发出多次读事件，即可能会有多次 select，所以需要 client 来保存状态
          Client client = new Client();
          client.buf = ByteBuffer.allocate(1024);
          CLIENT_MAP.put(socketChannel.getRemoteAddress(), client);
          logger.info("registered one connection");
          // if read IO
        } else if (selectionKey.isReadable()) {
          SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
          Client client = CLIENT_MAP.get(socketChannel.getRemoteAddress());
          while (socketChannel.read(client.buf) > 0) {
            client.buf.flip();
            // will not move on until read file name length successfully
            if (client.step == 0 && client.buf.remaining() >= 4) {
              client.fileNameLength = client.buf.getInt();
              logger.info("file name length：" + client.fileNameLength);
              client.step++;
            }
            // will not move on until read file name successfully
            if (client.step == 1 && client.buf.remaining() >= client.fileNameLength) {
              byte[] fileNameBytes = new byte[client.fileNameLength];
              client.buf.get(fileNameBytes);
              client.fileName = new String(fileNameBytes, UTF8);
              logger.info("file name：" + client.fileName);
              File file = new File(FILE_PATH + File.separator + client.fileName);
              if (!file.exists())
                file.createNewFile();
              client.fileChannel = client.getChannel(file);

              client.step++;
            }
            // will not move on until read file content length successfully
            if (client.step == 2 && client.buf.remaining() >= 8) {
              client.fileContentLength = client.buf.getLong();
              logger.info("file content length：" + client.fileContentLength);
              client.step++;
            }
            // will not move on until read file content successfully
            if (client.step == 3) {
              FileChannel fileChannel = client.fileChannel;
              fileChannel.write(client.buf);
              if (fileChannel.size() == client.fileContentLength) {
                logger.info("created file scuccessfully, close down all the resources");
                client.shutdown();
                selectionKey.cancel();
                CLIENT_MAP.remove(socketChannel.getRemoteAddress());
                break;
              }
            }
            // if pos < limit
            if (client.buf.hasRemaining()) {
              client.buf.compact();
            } else {
              client.buf.clear();
            }
          }
        }
        iterator.remove();
      }
    }
  } catch (IOException e) {
    e.printStackTrace();
  }
}
```

由于 NIO 传输是非阻塞、异步的，因此在传输过程中会出现“粘包”和“半包”问题。这里我创建了各个阶段的状态来判断是否 IO 完毕，但是这就要求了没有收到指定长度的内容就会一直自旋，可以增加一个变量用来控制自旋次数。后面章节会介绍根本性解决方案。
