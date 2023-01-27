# Java高并发核心编程

操作系统将内存（虚拟内存）划分为内核空间和用户空间。用户进程进行系统调用时必须切换为内核态。

用户程序进行 IO 的读写依赖于底层的 IO 读写，会用到`read`和`write`两大系统调用。`read`会把数据从内核缓冲区复制到应用程序的进程缓冲区，`write`会把数据从应用程序的进程缓冲区复制到操作系统的内核缓冲区。它们不负责数据在内核缓冲区和物理设备之间的交换。这个底层的读写交换操作是由操作系统内核（Kernel）完成。

## 1.四种主要的 IO 模型

阻塞 IO 指的是需要**内核 IO 操作彻底完成后**才返回到用户空间执行用户程序的操作指令。阻塞指的是用户程序的执行状态。

非阻塞 IO 指的是用户空间的程序不需要等待内核 IO 操作彻底完成，可以立即返回用户空间执行后续的指令。与此同时，内核会立即返回给用户一个 IO 状态值。

同步 IO 指用户空间是主动发起 IO 请求的一方，系统内核是被动接受方。

异步 IO 是指系统内核是主动发起 IO 请求的一方，用户空间是被动接受方。

内核 IO 操作：等待内核缓冲区数据 - 复制到用户缓冲区 - 复制完成

### 1.1 同步阻塞 IO

用户进程主动发起，需要等待内核 IO 操作彻底完成后才返回到用户空间的 IO 操作。优点是阻塞期间用户线程挂起，不会占用 CPU 资源，缺点是每个 IO 操作都需要创建一个线程，否则无法执行后续操作。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230108030608945-c56167152022504beeaaa241dd5c8ae8-e27dc2.png" alt="image-20230108030608945" style="zoom:67%;" />

### 1.2 同步非阻塞 IO

用户进程主动发起，不需要等待内核 IO 操作彻底完成就能立即返回用户空间的 IO 操作。优点是实时性较好，缺点是需要不断轮询内核，占用大量的 CPU 时间。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230108030634510-be4e92d736b0ba3b9a68981668f2e1a3-eaa6b2.png" alt="image-20230108030634510" style="zoom:67%;" />

> **注意**
>
> 当内核数据没有准备好时，用户线程的 IO 请求会立即返回。但是当内核数据到达后，用户线程发起系统调用==依然会阻塞==，内核会将内核缓冲区复制到用户缓冲区，然后内核返回结果。

### 1.3 IO 多路复用

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

### 1.4 异步 IO

用户线程通过系统调用向内核注册某个 IO 操作，内核在整个 IO 操作（包括数据准备、数据复制）完成后通知用户程序。理论上异步 IO 才是真正的异步输入输出，吞吐量高于 IO 多路复用模型。

> Windows 系统下通过 IOCP 实现了真正的异步 IO，在 Linux 系统下，异步 IO 模型在 2.6 版本才引入，JDK 对它的支持并不完善。因此目前高并发网络应用程序的开发大多采用 IO 多路复用模型。

## 2.Java NIO

OIO 是面向流的，NIO 是面向缓冲区的，可以随意读取 Buffer 中任意位置的数据。它使用了多路复用技术实现了非阻塞，即调用`read`方法会直接返回，底层是基于选择器的系统调用。

三个核心组件：

1. **Channel（通道）**

   在 OIO 中一个网络连接会关联两个流：输入流和输出流，在 NIO 中，一个网络连接使用一个通道表示，类似于 OIO 中两个流的结合体，可以从通道读取和写入数据。

2. **Buffer（缓冲区）**

   读取时将数据从通道中读取到缓冲区中，写入时将数据从缓冲区中写入到通道中。

3. **Selector（选择器）**

   用来监视文件描述符的就绪事件，通过选择器，一个线程可以查询多个通道的 IO 事件的就绪状态。

### 2.1 Buffer类

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

### 2.3 Channel类

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

