# 第05章_Decoder与Encoder核心组件

Netty 从底层 Java 通道读取 ByteBuf 二进制数据，传入 Netty 通道的流水线，随后开始入站处理。在入站处理过程中，需要将 ByteBuf 二进制类型解码成 Java POJO 对象。这个解码过程可以通过 Netty 的 Decoder 去完成。

相应的，在出站处理过程中，业务处理的结果需要从某个 Java POJO 对象编码为最终的 ByteBuf 二进制数据，然后通过底层 Java 通道发送到对端。在编码过程中可以使用 Netty 的 Encoder。

## 1.Decoder原理与实战

Netty 中的解码器是一个入站处理器，都直接或间接地实现了入站处理的超级接口`ChannelInboundHandler`。它能将上一个入站处理器传过来的输入数据进行解码或者格式转换，然后发送到下一个入站处理器。Netty 内置了`ByteToMessageDecoder`抽象类作为解码器的基类。

### 1.1 ByteToMessageDecoder处理流程

`ByteToMessageDecoder`是一个抽象类，实现了解码处理的基础逻辑和流程。它继承自`ChannelInboundHandlerAdapter`适配器，是一个入站处理器，用于完成从 ByteBuf 到 Java POJO 对象的解码功能。

首先，它会将上一站传过来的输入到 ByteBuf 中的数据进行解码，解码出一个`List<Object>对象列表`；然后，迭代这个列表，逐个将 Java POJO 对象传入下一个入站处理器。

由于它是一个抽象类，因此不能以实例化方式创建对象，需要依赖于它的具体实现。其中抽象的解码方法为`decode()`，子类需要实现这个方法定义将 ByteBuf 中的字节数据变成什么样的 Object 实例。

`ByteToMessageDecoder`使用了模版模式，具体过程是：

- 在`channelRead()`方法中调用`callDecode()`方法

- 通过调用子类的`decode()`方法对 ByteBuf 解码，由子类将解码结果放入`List<Object>`中
- 释放 ByteBuf 资源
- 将`List<Object>`中的元素**一个一个地传递**给下一个入站处理器

要实现一个解码器的流程大致如下：

- 继承`ByteToMessageDecoder`抽象类
- 实现基类的`decode()`抽象方法，定义将 ByteBuf 到目标 POJO 的解码逻辑
- 将解码后的 POJO 对象放入`decode()`方法的`List<Object>`实参中

### 1.2 自定义整数解码器

下面自定义一个整数解码器`Byte2IntegerDecoder`，用于将 ByteBuf 中的字节解码成整数类型。

```java
public class Byte2IntegerDecoder extends ByteToMessageDecoder {

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
	while (in.readableBytes() >= 4) {
	    int readInt = in.readInt();
	    Logger.info("解码出一个整数：" + readInt);
	    out.add(readInt);
	}
    }

}
```

要使用这个自定义的解码器，需要将其加入通道流水线中，还需要编写一个业务处理器，用于在读取解码后的 Java POJO 对象之后完成具体的业务逻辑。

接下来编写一个简单的配套处理器`IntegerProcessHandler`用于处理解码之后的整数。

```java
public class IntegerProcessHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
	Integer integer = (Integer) msg;
	Logger.info("读取出一个整数：" + integer);
	// 由于 msg 是整数，所以不存在释放资源的问题
    }

}
```

使用`EmbeddedChannel`编写一个测试用例

```java
public class Byte2IntegerDecoderTest {

    @Test
    void testByte2IntegerDecoder() {
	ChannelInitializer<EmbeddedChannel> initializer = new ChannelInitializer<EmbeddedChannel>() {
	    @Override
	    protected void initChannel(EmbeddedChannel ch) throws Exception {
		// 注意先后顺序：解码器在前
		ch.pipeline().addLast(new Byte2IntegerDecoder());
		ch.pipeline().addLast(new IntegerProcessHandler());
	    }
	};
	EmbeddedChannel channel = new EmbeddedChannel(initializer);
	for (int i = 0; i < 3; i++) {
	    ByteBuf buffer = Unpooled.buffer();
	    buffer.writeInt(i);
	    channel.writeInbound(buffer);
	}
    }

}
```

测试结果

```bash
[main|Byte2IntegerDecoder.decode] |>  解码出一个整数：0 
[main|IntegerProcessHandler.channelRead] |>  读取出一个整数：0 
[main|Byte2IntegerDecoder.decode] |>  解码出一个整数：1 
[main|IntegerProcessHandler.channelRead] |>  读取出一个整数：1 
[main|Byte2IntegerDecoder.decode] |>  解码出一个整数：2 
[main|IntegerProcessHandler.channelRead] |>  读取出一个整数：2 
```

最后说明一下，`ByteToMessageDecoder`传递给下一站的是解码之后的 Java POJO 对象，不是 ByteBuf 缓冲区。ByteBuf 的释放工作由父类`ByteToMessageDecoder`完成，它会调用`ByteBuf.release()`。如果后面还需要用到这个 Bytebuf 对象，可以调用`retain()`方法增加计数，使用完成后要注意释放，或传递到 TailContext。

### 1.3 ReplaingDecoder

使用上面的`Byte2IntegerDecoder`必须要对 ByteBuf 的可读字节长度进行检查，有足够的字节才能进行读取。我们可以使用 Netty 的`ReplayingDecoder`省去长度的判断。

`ReplayingDecoder`是`ByteToMessageDecoder`的子类，作用是：

- 在读取 ByteBuf 缓冲区前检查缓冲区是否有足够的字节
- 若有足够的字节则正常读取，否则停止解码

创建新的整数解码器`Byte2IntegerReplayDecoder`

```java
public class Byte2ReplayIntegerDecoder extends ReplayingDecoder {

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
	int i = in.readInt();
	Logger.info("解码出一个整数：" + i);
	out.add(i);
    }

}
```

`ReplayingDecoder`内部会将源 ByteBuf 装饰成了新的二进制缓冲区`ReplayingDecoderBuffer`并放入流水线，这个装饰器的特点是在缓冲区读不同类型数据之前会进行相应长度的判断：

```java
@Override
public int readInt() {
    checkReadableBytes(4);
    return buffer.readInt();
}
```

如果长度不够会抛出`ReplayError`。`ReplayingDecoder`重写了`callDecode`方法捕获 ReplayError 后会留着数据等待下一次 IO 事件到来时再读取。

此外，`ReplayingDecoder`会遍历源 ByteBuf 读取数据：

```java
@Override
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    // 包装 ByteBuf 成 ReplayingDecoderBuffer
    replayable.setCumulation(in);
    try {
        while (in.isReadable()) {
            // 记录当前读指针
            int oldReaderIndex = checkpoint = in.readerIndex();
            ...;
            try {
                // 内部调用 decode() 解码
                decodeRemovalReentryProtection(ctx, replayable, out);
                ...;
            } catch (Signal replay) {
				...;
                // 恢复读指针
                int checkpoint = this.checkpoint;
                if (checkpoint >= 0) {
                    in.readerIndex(checkpoint);
                } else {
                    // Called by cleanup() - no need to maintain the readerIndex
                    // anymore because the buffer has been released already.
                }
                // 结束当前循环，等待下一次 IO 读事件
                break;
            }
        }
    }
}
```

因此，`ReplayingDecoder`更重要的作用是用于分包传输的应用场景。

> **扩展：checkpoint**
>
> `ReplayingDecoder`通过`checkpoint`来保存`ReplayingDecoderBuffer`的读指针。在读取数据前先将当前读指针存入`checkpoint`，在读取过程中如果因为长度不够（可能在`decode`中多次读取了字节数据，但最后一次读取时长度不够，说明这次数据发生了分包）而抛出 ReplayError 异常，则`ReplayingDecoder`在捕获该异常后会将`ReplayingDecoderBuffer`的读指针恢复为`checkpoint`，下一次重新读取时还会从这个`checkpoint`开始。

### 1.4 案例：整数的分包解码器

底层通信协议是分包传输的，发送端发出去的包在传输过过程中会进行多次拆分和组装。如下图所示：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302111547631.png" alt="image-20230211154743609" style="zoom:50%;" />

在 Java OIO 流式传输中，接收端若读不到完整的信息则会一直阻塞（例如接收端调用`readLine()方法则会一直读到换行`），因此不会出现上述问题。但是在 NIO 中，保证一次性读取到完整的数据是个很大的问题。

在 Netty 中理论上可以使用`ReplayingDecoder`来解决上述问题。在进行数据解析时，如果发现当前 ByteBuf 中的可读数据不够，则`ReplayingDecoder`会一直等待直到可读数据足够。

这里先介绍一个简单点的例子——整数序列解码，并将解码出来的整数两两一组进行相加。要完成这个任务，需要用到`ReplayingDecoder`的`state`成员属性。该成员属性用于保存当前解码器在解码过程中所处的阶段。可通过有参构造器初始化`state`。

具体来说整个解码过程分成两个阶段，使用`state`属性来保存目前所处的阶段：

- 第一个阶段，提取第一个整数
- 第二个阶段，提取第二个整数，计算相加的结果，将结果输出

代码如下

```java
public class IntegerAddDecoder extends ReplayingDecoder<IntegerAddDecoder.PHASE> {

    enum PHASE {
	PHASE_1, PHASE_2
    }

    private int first, second;

    public IntegerAddDecoder() {
	// 初始化 state 属性为第一阶段
	super(PHASE.PHASE_1);
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
	switch (state()) {
	case PHASE_1:
	    first = in.readInt();
	    // 设置 state 为第二个阶段
	    state(PHASE.PHASE_2);
	    break;
	case PHASE_2:
	    second = in.readInt();
	    int sum = first + second;
	    out.add(sum);
	    state(PHASE.PHASE_1);
	    break;
	default:
	    break;
	}
    }

}

```

> **提示**
>
> 由于`checkpoint`的存在，下面的解码也可正常工作：
>
> ```java
> public class IntegerAddDecoder extends ReplayingDecoder<Integer> {
>     @Override
>     protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
> 	int first = in.readInt();
> 	int second = in.readInt();
> 	out.add(first + second);
>     }
> 
> }
> ```

### 1.5 案例：字符串的分包解码器

整数的长度是固定的，而字符串的长度是不固定的，所以在 Netty 中进行字符串的传输可以采用普通的 Head-Content 内容传输协议。

该协议的规则很简单：

- 在协议的 Head 部分放置字符串的字节长度，可以用一个整数类型来描述
- 在协议的 Content 部分放置字符串的字节数组

下面基于`ReplayingDecoder`实现自定义的字符串分包解码器：

```java
public class StringReplyDecoder extends ReplayingDecoder<StringReplyDecoder.PHASE> {

    enum PHASE {
	PHASE_1, PHASE_2
    }

    private int length;
    private byte[] bytes;

    public StringReplyDecoder() {
    // 初始化 state 属性为阶段 1
	super(PHASE.PHASE_1);
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
	switch (state()) {
    // 第一个阶段：解码出字符串长度
	case PHASE_1:
	    length = in.readInt();
	    bytes = new byte[length];
	    state(PHASE.PHASE_2);
	    break;
    // 第二个阶段：按照第一个阶段的字符串长度解码出字符串内容
	case PHASE_2:
	    in.readBytes(bytes, 0, length);
	    out.add(new String(bytes, "UTF-8"));
	    state(PHASE.PHASE_1);
	    break;
	default:
	    break;
	}
    }

}
```

为了配合字符串解码器，便写一个简单的业务处理器：

```java
public class StringProcessHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
	String s = (String) msg;
	Logger.info(s);
    }

}
```

测试

```java
public class StringReplyDecoderTester {

    private static final String CONTENT = "hello world!";
    private static final Charset UTF8 = Charset.forName("UTF-8");

    @Test
    void testStringReplyDecoder() {
	ChannelInitializer<EmbeddedChannel> initializer = new ChannelInitializer<EmbeddedChannel>() {
	    @Override
	    protected void initChannel(EmbeddedChannel ch) throws Exception {
		ch.pipeline().addLast(new StringReplyDecoder());
		ch.pipeline().addLast(new StringProcessHandler());
	    }
	};
	EmbeddedChannel channel = new EmbeddedChannel(initializer);

	byte[] contentBytes = CONTENT.getBytes(UTF8);
	Random random = new Random();
	for (int i = 0; i < 5; i++) {
	    int times = random.nextInt(3) + 1;
	    ByteBuf buf = Unpooled.buffer();
	    buf.writeInt(contentBytes.length * times);
	    // 为了测试，这里分批次发送
	    for (int k = 0; k < times; k++) {
		buf.writeBytes(contentBytes);
		// 增加一次 ByteBuf 的计数，避免发送后销毁
		buf.retain();
		channel.writeInbound(buf);
	    }
	    buf.release();
	}
    }

}
```

结果

```bash
[main|StringProcessHandler.channelRead] |>  hello world!hello world! 
[main|StringProcessHandler.channelRead] |>  hello world! 
[main|StringProcessHandler.channelRead] |>  hello world! 
[main|StringProcessHandler.channelRead] |>  hello world!hello world! 
[main|StringProcessHandler.channelRead] |>  hello world!hello world!hello world! 
```

虽然通过`ReplayingDecoder`解码器可以正确的解码分包后的 ByteBuf 数据包，但是开发中不建议使用这个类，原因如下：

- 不是所有的 ByteBuf 操作都被`ReplayingDecoderBuffer`装饰器支持，可能有些 ByteBuf 方法在`ReplayingDecoder`的`decode()`方法中会抛出 ReplayError 异常

- 在数据解码逻辑复杂的应用场景下，`ReplayingDecoder`在解码速度上相对较差

  因为当 ByteBuf 可读长度不够时会抛出 ReplayError 异常，`ReplayingDecoder`在捕获这个异常后，会把 ByteBuf 中的读指针还原到之前的读指针检查点（checkpoint），然后结束这次解析操作，等待下一次 IO 读事件。

  在网络条件糟糕是，一个数据包的解析逻辑会被反复执行多次，速度较差。因此`ReplayingDecoder`更多地应用于数据解析逻辑简单的场景。

接下来继承`ByteToMessageDecoder`基类实现一个定制的 Head-Content 协议字符串内容解码器：

```java
public class StringIntegerHeaderDecoder extends ByteToMessageDecoder {

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
	// 可读字节小于 4 说明消息头还没有读满
	if (in.readableBytes() < 4)
	    return;
	// 在真正读数据前，mark 当前的读指针
	in.markReaderIndex();
	// 读取消息头大小，会导致读指针发生变化
	int len = in.readInt();
	// 如果剩余长度不够消息体长度，则重制读指针，下次继续从消息头读取
	if (in.readableBytes() < len) {
	    in.resetReaderIndex();
	    return;
	}
	// 读到了消息体
	byte[] content = new byte[len];
	in.readBytes(content, 0, len);
	out.add(new String(content, "UTF-8"));
    }

}
```

表面上看`ByteToMessageDecoder`是无状态的，不像`ReplayingDecoder`需要使用状态位来保存当前的读取阶段。实际上其内部有一个二进制字节积累器`commulation`，用来保存没有解析完的二进制内容。所以`ByteToMessageDecoder`及其子类都是有状态的，其实例不能在通道间共享。每次初始化通道时，必须新创建一个`ByteToMessageDecoder`或者它的子类的实例。

### 1.6 MessageToMessageDecoder

当需要将一种 POJO 对象解码成另外一种 POJO 对象时可以继承一个新的 Netty 解码器基类`MessageToMessageDecoder<I>`。在继承它的时候需要明确的范性实参\<I\>，用于指定入站消息的 Java POJO 类型。

`MessageToMessageDecoder`同样使用了模版模式，需要子类实现`decode()`抽象方法。下面使用`MessageToMessageDecoder`实现一个整数到字符串转换的解码器：

```java
public class Integer2StringDecoder extends MessageToMessageDecoder<Integer> {
    @Override
    public void decode(ChannelHandlerContext ctx, Integer msg, List<Object> out) {
        out.add(String.valueOf(msg));
    }
}
```

同样的，在子类`decode()`方法处理完成后，父类会将 List 容器中的所有元素逐个发送给下一个入站处理器。

## 2.常用的内置Decoder

- `FixedLengthFrameDecoder`——固定长度数据包解码器

  适用场景：每个接收到的数据包长度都是固定的，例如 100 字节。它会把入站 ByteBuf 数据包拆分成一个个长度为 100 的数据包，然后发送到下一个入站处理器。

- `LineBasedFrameDecoder`——行分割数据包解码器

  适用场景：每个 ByteBuf 数据包使用换行符（或回车换行符）作为边界分隔符。它会使用换行分隔符把 ByteBuf 数据包分割成一个个完成的应用层 ByteBUf 数据包再发送到下一站。

- `DelimiterBasedFrameDecoder`——自定义分隔符数据包解码器

  它是`LineBasedFrameDecoder`的通用版本，可以自定义分隔符。

- `LengthFieldBasedFrameDecoder`——自定义长度数据包解码器

  这是一种基于灵活长度的解码器，在 ByteBUf 数据包中加入了一个长度字段，保存了原始数据包的长度，解码时会按照原始数据包长度进行提取。

### 2.1 LineBasedFrameDecoder

它的工作原理是：依次遍历 ByteBuf 数据包中的可读字节，判断在二进制字节流中是否存在换行符“\n”或者"\r\n"的字节码。如果有，就以此位置为结束位置，把从可读索引到结束位置之间的字节作为解码成功后的 ByteBuf 数据包。

它的构造器支持传入一个最大长度值，表示解码出来的 ByteBuf 能包含的最大字节数。如果连续读取到最大长度后依然没有发现换行符就会抛出异常。

代码演示

```java
public class NettyOpenBoxDecoder {

    private static final String SPLITER = "\r\n";
    private static final String CONTENT = "hello world!";
    private static final Charset UTF8 = Charset.forName("UTF-8");

    @Test
    void testLineBasedFrameDecoder() {
        ChannelInitializer<EmbeddedChannel> initializer = new ChannelInitializer<EmbeddedChannel>() {
            @Override
            protected void initChannel(EmbeddedChannel ch) throws Exception {
                ch.pipeline().addLast(new LineBasedFrameDecoder(1024));
                ch.pipeline().addLast(new Byte2StringDecoder());
                ch.pipeline().addLast(new StringProcessHandler());
            }
        };
        EmbeddedChannel channel = new EmbeddedChannel(initializer);
        Random random = new Random();
        for (int i = 0; i < 5; i++) {
            int times = random.nextInt(3) + 1;
            ByteBuf buf = Unpooled.buffer();
            for (int k = 0; k < times; k++) {
            	buf.writeBytes(CONTENT.getBytes(UTF8));
            }
            buf.writeBytes(SPLITER.getBytes(UTF8));
            // 发送一个回车换行符作为包结束符
            channel.writeInbound(buf);
        }
        channel.close();
    }

    class Byte2StringDecoder extends ByteToMessageDecoder {
        @Override
        protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
            byte[] bytes = new byte[in.readableBytes()];
            in.readBytes(bytes);
            out.add(new String(bytes, UTF8));
        }
    }

}
```

结果

```bash
[main|StringProcessHandler.channelRead] |>  hello world!hello world!hello world! 
[main|StringProcessHandler.channelRead] |>  hello world! 
[main|StringProcessHandler.channelRead] |>  hello world!hello world! 
[main|StringProcessHandler.channelRead] |>  hello world!hello world!hello world! 
[main|StringProcessHandler.channelRead] |>  hello world!hello world!hello world! 
```

### 2.2 DelimiterBasedFrameDecoder

`DelimiterBasedFrameDecoder`不仅可以使用换行符，还可以使用其他特殊字符作为数据包的分隔符，例如制表符“\t”。

其构造方法如下：

```java
public DelimiterBasedFrameDecoder(
    int maxFrameLength,      // 解码的数据包的最大长度
    Boolean stripDelimiter,  // 解码后的数据包是否去掉分隔符，一般选择是
    ByteBuf delimiter        // 分隔符
){}
```

其使用方法与`LineBasedFrameDecoder`是一样的：

```java
public class NettyOpenBoxDecoder {

    private static final String SPLITER2 = "\t";
    private static final String CONTENT = "hello world!";
    private static final Charset UTF8 = Charset.forName("UTF-8");

    @Test
    void testDelimiterBasedFrameDecoder() {
	ChannelInitializer<EmbeddedChannel> initializer = new ChannelInitializer<EmbeddedChannel>() {
	    final ByteBuf DELIMITER = Unpooled.copiedBuffer(SPLITER2.getBytes(UTF8));

	    @Override
	    protected void initChannel(EmbeddedChannel ch) throws Exception {
            ch.pipeline().addLast(new DelimiterBasedFrameDecoder(1024, true, DELIMITER));
            ch.pipeline().addLast(new Byte2StringDecoder());
            ch.pipeline().addLast(new StringProcessHandler());
	    }
	};
	EmbeddedChannel channel = new EmbeddedChannel(initializer);
	Random random = new Random();
	for (int i = 0; i < 5; i++) {
	    int times = random.nextInt(3) + 1;
	    ByteBuf buf = Unpooled.buffer();
	    for (int k = 0; k < times; k++) {
			buf.writeBytes(CONTENT.getBytes(UTF8));
	    }
	    buf.writeBytes(SPLITER2.getBytes(UTF8));
	    channel.writeInbound(buf);
	}
	channel.close();
    }
}
```

### 2.3 LengthFieldBasedFrameDecoder

`LengthFieldBasedFrameDecoder`最为复杂也最为常用，可以称它为“长度字段数据包解码器”。普通的基于 Head-Content 协议的内容传输尽量此解码器。

下面是`LengthFieldBasedFrameDecoder`的一个构造器事例：

```java
public LengthFieldBasedFrameDecoder (
	int maxFrameLength,
    int lengthFieldOffset,
    int lengthFieldLength,
    int lengthAdjustment,
    int initialBytesToStrip
){}
```

- `maxFrameLength`：发送的数据包的最大长度

- `lengthFieldOffset`：长度字段偏移量，指长度字段位于整个数据包内部字节数组的下标索引值

- `lengthFieldLength`：长度字段所占的字节数

- `lengthAdjustment`：长度的调整值，实际内容字段相对于长度字段的偏移量

  如果传输协议中包含了长度字段、协议版本号、魔数等，那么解码时就需要进行长度调整。长度调整值的计算公式为：内容字段偏移量 - 长度字段偏移量 - 长度字段的字节数。

- `initialBytesToStrop`：丢弃的起始字节数

  在有效数据字段 Content 前面如果还有一些其他字段的字节，作为最终的解析结果可以丢弃。

简单的`LengthFieldBasedFrameDecoder`示例

```java
public class NettyOpenBoxDecoder {

    private static final String CONTENT = "hello world!";
    private static final Charset UTF8 = Charset.forName("UTF-8");

    @Test
    void testLengthFieldBasedFrameDecoder() {
	ChannelInitializer<EmbeddedChannel> initializer = new ChannelInitializer<EmbeddedChannel>() {

	    @Override
	    protected void initChannel(EmbeddedChannel ch) throws Exception {
            // 最大长度：1024
            // 长度字段偏移量：0，即长度字段处于数据包的起始位置
            // 长度字段内容长度：4，int 类型占用 4 个字节
            // 长度调整值：0，即实际内容字段紧挨着长度字段
            // 丢弃字段长度：4，即最终内容抛弃最前面的 4 个字节数据（这里指长度字段数据）
            ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(1024, 0, 4, 0, 4));
            ch.pipeline().addLast(new Byte2StringDecoder());
            ch.pipeline().addLast(new StringProcessHandler());
	    }
	};
	EmbeddedChannel channel = new EmbeddedChannel(initializer);
	Random random = new Random();
	byte[] contentBytes = CONTENT.getBytes(UTF8);
	for (int i = 0; i < 10; i++) {
	    int times = random.nextInt(3) + 1;
	    ByteBuf buf = Unpooled.buffer();
	    // 先写入 4 个字节的长度
	    buf.writeInt(contentBytes.length * times);
	    for (int k = 0; k < times; k++) {
            // 再写入内容
            buf.writeBytes(contentBytes);
	    }
	    channel.writeInbound(buf);
	}
	channel.close();
    }
}

```

结果

```bash
[main|StringProcessHandler.channelRead] |>  hello world!hello world!hello world! 
[main|StringProcessHandler.channelRead] |>  hello world!hello world! 
[main|StringProcessHandler.channelRead] |>  hello world!hello world! 
[main|StringProcessHandler.channelRead] |>  hello world! 
...
```

### 2.4 案例：多字段Head-Content协议数据包解析

在实际使用过程中，除了长度和内容，在数据包中还可能包含其他字段，例如协议版本号：

```bash
 - 长度字段，4 个字节    - 版本字段，2 个字节     - 内容字段，52 个字节
|                     |                     |
52                    10                    hello world...
```

此时参数可设置为：

```java
new LengthFieldBasedFrameDecoder(1024, 0, 4, 2, 6);
```

- 长度调整值：2，即内容偏移长度字段 2 个字节
- 丢弃字节长度：6，即最终内容丢弃长度字段和版本字段

最终输出结果和案例 2.3 一致。

如果将协议设计得再复杂一点：将 2 字节的版本字段放在最前面，在长度字段后面加上 4 字节的魔数，内容大致如下：

```bash
 - 版本字段，2 个字节   - 长度字段，4 个字节    - 魔数，4 个字节   - 内容字段，52 个字节
|                    |                    |                |
10                   52                   10               hello world!...
```

此时参数可设置为：

```java
new LengthFieldBasedFrameDecoder(1024, 2, 4, 4, 10);
```

- 长度字段偏移：2
- 长度字段长度：4
- 长度字段调整：4
- 丢此字节长度：10

## 3.Encoder原理与实战

在 Netty 的业务处理完成后，业务处理的结果往往是某个 Java POJO 对象需要编码成最终的 ByteBuf 二进制类型，通过流水线写入底层的 Java 通道，这时就需要用到 Encoder。

首先，编码器是一个出站处理器，负责处理出站数据；其次，编码器将上一个出站处理器传过来的数据进行编码或者格式转换后，传递给下一个出站处理器。

最后一个出站处理器需要将数据编码成 ByteBuf 类型。

### 3.1 MessageToByteEncoder

`MessageToByteEncoder`的功能是将一个 Java POJO 对象编码成一个 ByteBuf 数据包。同样的，需要子类实现`encode()`方法完成具体的编码逻辑。

下面实现一个整数编码器，用于将 Java 整数编码成二进制 ByteBuf 数据包：

```java
public class Integer2ByteEncoder extends MessageToByteEncoder<Integer> {
    @Override
    public void encode(ChannelHandlerContext ctx, Integer msg, ByteBuf out) {
        out.writeInt(msg);
    }
}
```

在继承`MessageToByteEncoder`时需要带上范性实参，即标识编码前的 Java POJO 输入类型。编码完成后需要将 msg 写入 Out 实参中。最后由`MessageToByteEncoder`将输出的 ByteBuf 数据包发送到下一站。

测试

```java
@Test
void testInteger2ByteEncoder() {
    ChannelInitializer<EmbeddedChannel> initializer = new ChannelInitializer<EmbeddedChannel>() {
        @Override
        protected void initChannel(EmbeddedChannel ch) throws Exception {
        ch.pipeline().addLast(new Integer2ByteEncoder());
        }
    };
    EmbeddedChannel channel = new EmbeddedChannel(initializer);
    for (int i = 0; i < 3; i++)
        channel.write(i);
    channel.flush();
    ByteBuf outBuf = (ByteBuf) channel.readOutbound();
    while (outBuf != null) {
        Logger.info(outBuf.readInt());
        outBuf = (ByteBuf) channel.readOutbound();
    }
}
```

结果

```bash
[main|NettyOpenBoxDecoder.testInteger2ByteEncoder] |>  0 
[main|NettyOpenBoxDecoder.testInteger2ByteEncoder] |>  1 
[main|NettyOpenBoxDecoder.testInteger2ByteEncoder] |>  2 
```

### 3.2 MessageToMessageEncoder

如果需要将一种 POJO 对象编码成另一种 POJO 对象，可以继承`MessageToMessageEncoder`并实现`encode()`方法。在`encode()`实现方法中，编码完成后将结果加入 list 输出容器即可。基类会对这个 list 输出容器中的所有元素进行迭代，将元素逐个发送给下一站。它有一个范型参数，用来制定源数据类型。

下面是一个从字符串到整数的编码器，将字符串中的所有数字提取出来，然后输出到下一站：

```java
public class String2IntegerEncoder extends MessageToMessageEncoder<String> {

    @Override
    protected void encode(ChannelHandlerContext ctx, String msg, List<Object> out) throws Exception {
	Logger.info(msg);
	char[] charArray = msg.toCharArray();
	for (char a : charArray) {
	    // 48 是 0 的编码，57 是 9 的编码
	    if (a >= 48 && a <= 57)
		// 将 char 类型转换为 int 类型
		out.add(Character.getNumericValue(a));
	}
    }

}
```

测试

```java
@Test
void testString2IntegerEncoder() {
ChannelInitializer<EmbeddedChannel> initializer = new ChannelInitializer<EmbeddedChannel>() {
    @Override
    protected void initChannel(EmbeddedChannel ch) throws Exception {
    ch.pipeline().addLast(new Integer2ByteEncoder());
    ch.pipeline().addLast(new String2IntegerEncoder());
    }
};
EmbeddedChannel channel = new EmbeddedChannel(initializer);

for (int i = 0; i < 5; i++) {
    String s = "hello" + i;
    channel.write(s);
}
channel.flush();
ByteBuf outbound = (ByteBuf) channel.readOutbound();
while (null != outbound) {
    Logger.info(outbound.readInt());
    outbound = (ByteBuf) channel.readOutbound();
}
}
```

结果

```bash
[main|String2IntegerEncoder.encode] |>  hello0 
[main|String2IntegerEncoder.encode] |>  hello1 
[main|String2IntegerEncoder.encode] |>  hello2 
[main|String2IntegerEncoder.encode] |>  hello3 
[main|String2IntegerEncoder.encode] |>  hello4 
[main|NettyOpenBoxDecoder.testString2IntegerEncoder] |>  0 
[main|NettyOpenBoxDecoder.testString2IntegerEncoder] |>  1 
[main|NettyOpenBoxDecoder.testString2IntegerEncoder] |>  2 
[main|NettyOpenBoxDecoder.testString2IntegerEncoder] |>  3 
[main|NettyOpenBoxDecoder.testString2IntegerEncoder] |>  4 
```

## 4.解码器和编码器的结合

前面讲到的编码器和解码器是分开实现的，具有相反逻辑的配套编码器和解码器再加入通道的流水线时常常需要分两次添加，使用`Codec`（编解码器）可将相互配套逻辑的编码器和解码器放在一个类中，加入流水线时也只需要加入一次。

### 4.1 ByteToMessageCodec

完成 POJO 到 ByteBuf 数据包的编解码器基类为`ByteToMessageCodec<I>`，它是一个抽象类。从功能上说，继承`ByteToMessageCodec<I>`就等同于继承了`ByteToMessageDecoder`和`MessageToByteEncoder`，因此需要同时实现`encode()`和`decode()`两个抽象方法。

下面是一个整数到字节、字节到整数的编解码器：

```java
public class Byte2IntegerCodec extends ByteToMessageCodec<Integer> {

    @Override
    protected void encode(ChannelHandlerContext ctx, Integer msg, ByteBuf out) throws Exception {
	out.writeInt(msg);
	Logger.info("encode int: " + msg);
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
	if (in.readableBytes() >= 4) {
	    int readInt = in.readInt();
	    out.add(readInt);
	    Logger.info("decode int: " + readInt);
	}
    }

}
```

测试

```java
@Test
void testByte2IntegerCodec() {
    ChannelInitializer<EmbeddedChannel> initializer = new ChannelInitializer<EmbeddedChannel>() {
        @Override
        protected void initChannel(EmbeddedChannel ch) throws Exception {
        ch.pipeline().addLast(new Byte2IntegerCodec());
        ch.pipeline().addLast(new IntegerProcessHandler());
        }
    };
    EmbeddedChannel channel = new EmbeddedChannel(initializer);
    for (int i = 0; i < 4; i++) {
        byte[] bytes = intToBytes(i);
        // 按字节逐个发送出去
        for (int j = 3; j >= 0; j--) {
        ByteBuf buffer = Unpooled.buffer();
        buffer.writeByte(bytes[j]);
        channel.writeInbound(buffer);
        }
    }
    ByteBuf outbound = (ByteBuf) channel.readOutbound();
    while (null != outbound) {
        Logger.info(outbound.readInt());
        outbound = (ByteBuf) channel.readOutbound();
    }
}

public static byte[] intToBytes(int value) {
	byte[] src = new byte[4];
	// 每次得到低 8 位
	src[0] = (byte) (value & 0xFF);
	src[1] = (byte) ((value >> 8) & 0xFF);
	src[2] = (byte) ((value >> 16) & 0xFF);
	src[3] = (byte) ((value >> 24) & 0xFF);
	return src;
    }

class IntegerProcessHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        // 必须要调用 flush 方法，不然外面无法读取到出站的数据
		ctx.channel().writeAndFlush(msg);
    }

}
```

结果

```bash
[main|Byte2IntegerCodec.decode] |>  decode int: 0 
[main|Byte2IntegerCodec.encode] |>  encode int: 0 
[main|Byte2IntegerCodec.decode] |>  decode int: 1 
[main|Byte2IntegerCodec.encode] |>  encode int: 1 
[main|Byte2IntegerCodec.decode] |>  decode int: 2 
[main|Byte2IntegerCodec.encode] |>  encode int: 2 
[main|Byte2IntegerCodec.decode] |>  decode int: 3 
[main|Byte2IntegerCodec.encode] |>  encode int: 3 
[main|Byte2IntegerDecoderTest.testByte2IntegerCodec] |>  0 
[main|Byte2IntegerDecoderTest.testByte2IntegerCodec] |>  1 
[main|Byte2IntegerDecoderTest.testByte2IntegerCodec] |>  2 
[main|Byte2IntegerDecoderTest.testByte2IntegerCodec] |>  3 
```

对于 POJO 之间的编码和解码，Netty 提供了`MessageToMessageCodec`用于完成 POJO-TO-POJO 的双向转换，其原理大致相同，这里省略具体实现。

### 4.2 CombinedChannelDuplexHandler

前面的编码器和解码器通过继承的方式实现，因此必须强制性地放在同一个类中，在只需要编码或者解码操作的流水线上就不太合适。除了继承方式，还可以通过**组合**的方式实现。

下面通过示例演示如何通过组合器`CombinedChannelDuplexHandler`将整数解码器和整数编码器组合起来：

```java
public class IntegerDuplexHandler extends CombinedChannelDuplexHandler<Byte2IntegerDecoder, Integer2ByteEncoder> {

    public IntegerDuplexHandler() {
	super(new Byte2IntegerDecoder(), new Integer2ByteEncoder());
    }
}
```

接下来只需要把`CombinedChannelDuplexHandler`加入流水线即可：

```java
ChannelInitializer<EmbeddedChannel> initializer = new ChannelInitializer<EmbeddedChannel>() {
    @Override
    protected void initChannel(EmbeddedChannel ch) throws Exception {
    ch.pipeline().addLast(new IntegerDuplexHandler());
    ch.pipeline().addLast(new IntegerProcessHandler());
    }
};
```

这样就可以分开实现解码器和编码器。
