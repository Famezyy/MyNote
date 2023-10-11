# 第08章_HTTP原理与Web服务器实战

## 8.1高性能Web应用架构

### 1.十万级并发的Web应用架构

QPS 在十万每秒的 Web 应用架构大致如下：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302242222097.png" alt="image-20230224222248617" style="zoom: 33%;" />

主要包括客户端层、接入层、服务层，重点是**接入层**和**服务层**。

- **服务层**

  在 Spring Cloud 微服务技术成熟之前，服务层主要是通过 Tomcat 集群部署向外提供服务；在微服务技术成为主流后，服务层主要是微服务 Provider 实例，并通过内部网关向外提供统一的访问服务。

- **接入层**

  接入层可以理解为客户端层与服务层之间的一个反向代理层，利用高性能的 Nginx 来做反向代理：

  1. Nginx 将客户端请求分发给上游的多个 Web 服务；Nginx 向外暴露一个外网 IP，Nginx 和内部 Web 服务之间使用内网访问
  2. Nginx 需要保障负载均衡，并通过 Lua 脚本具备动态伸缩、动态增加 Web 服务节点的能力
  3. Nginx 需要保障系统的高可用，任何一台 Web 服务节点挂了，Nginx 都可以将流量迁移到其他 Web 服务节点上


### 2.扩展：Nginx

Nginx 的原理与 Netty 很像，也应用了 Reactor 模式。Nginx 执行过程中主要包括一个 Master 进程和 n（大于等于 1）个 Worker 进程，所有进程都是单线程的。Nginx 使用了多路复用和事件通知。其中 Master 进程用于接收来自外界的信号，并给 Worker 进程发送信号，同时监控 Worker 进程的工作状态。Worker 进程则是外部请求真正的处理器，每个 Worker 请求相互独立且平等的竞争来自客户端的请求。

因为 Nginx 使用了 Reactor 模式，所以在处理高并发请求时内存消耗非常小。在 30000 并发连接下，开启的 10 个 Nginx 进程才消耗 150（15 * 10）MB 内存。

与 Nginx 类似，同样比较有名的 Web 服务器还有 Apache HTTP Server（纯 Java 实现）。该服务器在处理并发连接时会为每个连接建立一个单独的进程或线程，并且在网络输入/输出操作时阻塞。该阻塞式的 IO 将导致内存和 CPU 被大量消耗，因为新建一个单独的进程或线程需要准备新的运行时环境，包括堆内存和栈内存的分配，以及新的执行上下文，这些操作也会导致多余的 CPU 开销。最终会由于过多的上下文切换而导致服务器性能变差。因此接入层的反向代理服务器原则上需要使用高性能的 Nginx 而不是 Apache HTTP Server。

尽管单体的 Nginx 比较稳定，在长时间运行的情况下，还是存在有可能崩溃的情况，此时可以使用 Nginx + KeepAlived 组合模式，具体如下：

1. 使用两台以上的 Nginx 组成一个集群，分别部署上 KeepAlived，设置成相同的虚 IP 供下游访问
2. 当一台 Nginx 挂了，KeepAlived 能够探测到并将流量自动迁移到另一台 Nginx 上，整个过程对下游调用方透明

### 3.千万级高并发的Web应用架构

QPS 在百万级甚至千万级的 Web 应用架构大致如下：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302242242902.png" alt="image-20230224224257885" style="zoom: 33%;" />

主要包括客户端层、负载均衡层、接入层、服务层，重点是客户端层和负载均衡层。

在客户端层，需要在 DNS 服务器上使用负载均衡机制。DNS 负载均衡具体来说就是在 DNS 服务器中配置多个 A 记录，如下所示：

|     DNS 服务器      |     示例     |
| :-----------------: | :----------: |
| `www.test.com` IN A | 114.100.80.1 |
| `www.test.com` IN A | 114.100.80.2 |
| `www.test.com` IN A | 114.100.80.3 |

在一个域名下添加多个 IP，由 DNS 域名服务器进行多个 IP 之间的负载均衡，甚至 DNS 服务器可以按照就近原则为用户返回最近的服务器 IP 地址。

DNS 负载均衡也有不少缺点：

- 通常无法动态调整主机地址权重（也有支持权重配置的 DNS 服务器），如果多台主机性能差异较大，则不能很好地负载均衡
- DNS 服务器通常会缓存查询响应，以便更快地向用户提供查询服务。在某台主机宕机时，即使第一时间移除服务器 IP 也无济于事

由于 DNS 负载均衡无法满足高可用性要求，因此通常仅仅被用于客户端层的简单复杂均衡。还需要在客户端和接入层之间引入一个专门的负载均衡层，该层通过 LVS + KeepAlived 组合模式达到高可用和负载均衡的目的。

具体方案如下：

1. 使用两台或以上 LVS 组成一个集群，分别部署上 KeepAlived，设置成相同的虚 IP 供下游访问。KeepAlived 对 LVS 负载调度器实现健康监控、热备切换，具体来说，就是对服务器池中各个节点进行健康检查，自动移除时效节点，恢复后再重新加入，从而保证 LVS 高可用
2. 在 LVS 系统上，配置多个接入层 Nginx 服务器集群，由 LVS 完成高速的请求分发和接入层的负载均衡

### 4.LVS

LVS（Linux VIrtual Server）是一个虚拟的 Linux 服务器，该项目在 1998 年 5 月由章文嵩博士成立，是国内最早出现的自由软件项目之一。LVS 目前是 Linux 标准内核的一部分，从 Linux 2.4 内核之后，无需任何布丁，可以直接使用 LVS 提供的各种功能。

LVS 常常使用直接路由方式（DR）进行负载均衡，数据在分发过程中不修改 IP 地址，只修改 MAC 地址，由于实际处理请求的真实物理 IP 地址和数据请求目的 IP 地址一致，因此响应数据包可以不需要通过 LVS 负载均衡服务器进行地址转换，而是直接返回给用户浏览器，避免 LVS 负载均衡服务器网卡带宽成为瓶颈，此种方式又称为三角传输模式，具体如下：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302242302802.png" alt="image-20230224230239784" style="zoom: 33%;" />

使用三角传输模式的链路层负载均衡是目前大型网站使用最广泛的一种负载均衡手段。目前，LVS 是 Linux 平台上最好的三角传输模式软件负载均衡开源产品。除了软件产品外，还可以使用性能更好的硬件产品（如 F5）。

> **说明**
>
> LVS 和 Nginx 都具备负载均衡能力，他们的区别是 Ngxin 主要用于 4、7 层的负载均衡，平时使用 Nginx 进行的 Web Server 负载均衡就属于 7 层负载均衡；LVS 主要用于 2、4 层的负载均衡，但是出于性能的原因，LVS 更多用于 2 层（数据链路层）负载均衡。
>
> 2、4、7 层负载均衡
>
> - 2 层（OSI 模型的数据链路层）负载均衡：主要根据报文中的链路内容（如 MAC 地址等）在多个上游服务器之间选择一个 RS（Real Server），然后进行报文的处理和转发，从而实现负载均衡
> - 4 层（OSI 模型的传输层）负载均衡：主要通过修改报文中的目标 IP 地址和端口在多个上游 TCP/UDP 服务器之间选择一个 RS，然后进行报文转发实现负载均衡
> - 7 层（OSI 模型的应用层）负载均衡：主要根据报文中的应用层内容（如 HTTP 协议 URI、Cookie 信息、虚拟主机 HOST 名称等）在多个上游应用层服务器（如 HTTP Web 服务器）之间选择一个 RS，然后进行报文转发从而实现负载均衡

LVS 转发分为 NAT 模式（4 层负载均衡）和 DR 模式（2 层负载均衡）。

- **NAT 模式**

  NAT 包括目标地址转换（DNAT）和源地址转换（SNAT）。当包到达 LVS 时，LVS 需要做目标地址转换（DNAT）：将目标 IP 改为 RS 的 IP，RS 在接收到数据包以后，仿佛是客户端直接发给他的一样；RS 处理完返回响应时，源 IP 是 RS 的 IP，目标 IP 是客户端的 IP，这时 LVS 需要做源地址转换（SNAT），将包的源地址改为 VIP（对外的 IP），这样这个包对客户端看起来就仿佛是 LVS 直接返回给它的。

- **DR 模式**

  也叫直接路由、三角传输模式。**DR 模式下需要 LVS 和 RS 集群绑定在同一个虚拟 IP（VIP）上**，与 NAT 不同的是，请求由 LVS 接受，处理后由 RS 直接返回给用户，响应返回的时候不经过 LVS。

  一个请求过来时，LVS 只需要将网络帧的 MAC 地址修改为某一台 RS 的 MAC，该包就会被转发到相应的 RS 处理，此时的源 IP 和目标 IP 都没变，RS 收到 LVS 转发来的包时，链路层发现 MAC 是自己的，网络层发现 IP 是自己的，于是这个包被合法地接收，RS 感知不到 LVS 的存在。当 RS 返回响应时，直接向源 IP（客户端 IP）返回即可，不再经过 LVS 转发。

  这里要注意，RS 的 Loopback 口需要和 LVS 设备上存在相同的 VIP 地址，这样响应才能直接返回到客户端。

## 8.2 详解HTTP应用层协议

HTTP（Hyper Text Transfer Protocol，超文本传输协议），是一个基于请求与响应，无状态的应用层的协议，所有的 WWW 文件必须遵守这个标准，设计 HTTP 的初衷是为了提供一种发布和接收 HTML 页面的方法。

TCP/IP 与 HTTP 协议的关系大致可以描述为在传输数据时，应用程序之间可以只使用 TCP/IP（传输层）协议，如果没有应用层，应用程序便无法识别数据内容。如果想要使传输的数据有意义，必须使用应用层协议。应用层协议有很多，比如 HTTP、FTP、TELNET 等，也可以自定义应用层协议。

### 1.HTTP简介

HTTP 是一个属于应用层的面向对象的协议，适用于分布式超媒体信息系统，是现在应用最为广泛的一种网络协议。1960 年美国人 Ted Nelson 构思了一种痛殴计算机处理文本信息的方法，并成为超文本（Hyper Text），称为 HTTP 超文本传输协议标准架构的发展根基。最终，万维网协会（World Wide Web Consortium）和互联网工程工作小组（Internet Engineering Task Force）共同合作研究 HTTP，最终发布了一系列的 RFC 文档，其中最著名的 RFC 2616 定义了 HTTP 1.1 协议。

HTTP 的主要特点如下：

- 支持客户端/服务端模式

- 简单快速

  客户向服务器请求服务时，只需传送请求方法和路径。请求方法常用的有`GET`、`HEAD`、`POST`，且每种方法规定了客户与服务器联系的类型不同。HTTP 简单，使得 HTTP 服务器的程序规模较小，因此通信速度快。

- 灵活

  HTTP 允许传输任意类型的数据对象，数据的类型由`Content-Type`指定。

- 无连接

  每次连接至处理一个请求，服务器处理完客户的请求并收到客户的应答后即断开连接。

- 无状态

  协议对于书屋处理没有记忆能力。如果后续处理需要前面的信息，则必须重传，这样会导致每次连接传送的数据量增大。

### 2.请求URL

使用 HTTP 通信时，通过使用 Web 浏览器、网络爬虫或者其他的工具，客户端发起一个到服务器上指定端口（默认 80）的 HTTP 请求，在客户端和源服务器中间可能存在多个中间层，比如代理、网关或者隧道。

通常，由 HTTP 客户端发起一个请求，建立一个到服务器指定端口的 TCP 连接。同时 HTTP 服务器监听相应的端口，一旦服务器收到请求，就会返回一个状态行（比如“HTTP/ 1.1 200 OK”）和消息响应。消息的消息体可能是请求的文件、错误消息或者其他信息。

> **扩展：为什么不使用 UDP？**
>
> 打开一个网页必需传送很多数据，而 TCP 提供传输控制，按顺序组织数据、纠正错误。

通过 HTTP 协议请求的资源由统一资源表示符（Uniform Resource Identifier，URI）来标识。在 Java 中，更多的是 URI 的子类统一资源定位符 URL（URL 是一种特殊类型的 URI，包含了用于查找某个资源的足够信息），格式为`http://host[":"port][abs_path]`。

在 URL 中，`http`表示要通过 HTTP 来定位网络资源；`host`表示合法的 internet 主机域名或者 IP 地址；`post`指定一个端口号，为空则使用默认端口 80；`abs_path`指定请求资源的 URI；如果 URL 中没有给出`abs_path`，则当请求 URI 时，就必须以“/”的形式给出，通常浏览器会自动帮我们完成。例如在浏览器地址栏输入`www.test.com`，浏览器会自动转换为`http://www.test.com/`

### 3.请求报文

HTTP 请求由三部分组成，分别是请求行、请求头、请求体，一般也会将 HTTP 的请求行和请求头统一称为请求首部。如下图所示：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302250049841.png" alt="image-20230225004958828" style="zoom: 33%;" />

#### 3.1 请求行

HTTP 请求的请求行包含请求方法、URL 地址、协议名称和版本号。

它以一个方法（Method）符号开头，以空格分开，后面跟着请求的 URI 和协议的版本，格式为`Method Request-URI HTTP-Version CRLF`。例如`POST http://www.test.com:8080/test HTTP/1.1`。

其中`Method`标识请求方法。HTTP/1.1 定义了 8 种请求方法：GET、POST、PUT、DELETE、PATCH、HEAD、OPTIONS、TRACE，其中最常用的是 GET 和 POST。如果是 RESTful 接口，一般会用到 GET、POST、DELETE 和 PUT。

`Request-URI`是一个统一资源表示符。它和请求头的 Host 属性组成完整的请求 URL。URL 可以以 key-value 的方式传递请求参数。

`HTTP-Version`表示请求的 HTTP 协议版本；CRLF 表示回车和换行（除了作为结尾的 CRLF 外，不允许出现单独的 CR 或 LF 字符）。

#### 3.2 请求头

包含若干头部的字段，大多请求头不是必须的，但对于 POST 请求来说，`Content-Length`是必须存在的。常见的请求头字段如下：

- `Accept`：客户端可接受的 MME 类型

- `Accept-Charset`：客户端可接受的字符集

- `Accept-Encoding`：客户端能够进行解码的数据解码方式，比如 gzip，Servlet 能够向支持 gzip 的客户端返回经 gzip 编码的 HTML 页面，许多情况下可以减少 5～10 倍的下载时间

- `Accept-Language`：客户端希望的语言种类，当服务器能够提供一种以上的语言版本时需要设置

- `Authorization`：用于设置用户身份信息，如果使用`Authorization`的方式进行认证，则每次都要将认证的身份信息（如令牌）放到`Authorization`头部

- `Content-Length`：表示请求消息正文的长度

- `Host`：客户端通过这个头部信息告诉服务器想访问的主机名，Host 头字段指定请求资源的主机和端口号，必须表示请求 URL 的原始服务器或网关的位置，HTTP/1.1 请求必须包含主机头字段，否则系统会以 400 状态码返回

- `If-Modified-Since`：客户端通过这个头部信息告诉服务器资源的缓存时间，只有当请求的内容在指定的时间后又经过修改才返回，否则返回 304（Not Modified） 应答

- `Referer`：客户端通过这个头部字段告诉服务器它是从哪个资源来访问服务器的（放盗链），Referer 包含一个 URL，表示用户从该 URL 代表的页面出发访问当前请求的页面

- `User-Agent`：包含发出请求的用户信息

- `Cookie`：客户端通过这个头部信息往服务器发数据，最重要的请求头之一

- `Pragma`：值为“no-cache”，表示服务器必须返回一个刷新后的文档，如果服务器是代理服务器而且已经有了页面的本地缓存副本，则需要进行本地缓存副本的刷新

- `From`：值为请求发送者的 email 地址，由一些特殊的 Web 客户程序使用，HTTP 客户端不会用到

- `Connection`：描述请求完成后是断开连接还是继续保持连接，如果是`Keep-Alive`或者客户端使用的是 HTTP 1.1（默认进行持久连接），它就可以利用持久连接的优点，当页面包含多个元素时（例如 Applet、图片等），会显著减少下载所需要的时间，持久连接也需要服务端进行配合，服务端需要在应答中发送一个`Content-Length`头，发送响应内容的大小

- `Range`：用于请求 URL 资源的部分内容，单位是 byte（字节），并从 0 开始。如果请求头携带了 Range 信息，则表示客户端需要进行分批下载或者分段传输。如果服务端支持分批下载，则服务器会返回状态码 206（Partial Content）以及该部分内容。如果服务器不支持分批下载，则服务器会返回整个资源的大小以及状态码 200.不同的请求范围对应的 Range 头部值如下：

  |      Range 头部值       |          实例          |
  | :---------------------: | :--------------------: |
  |     表示头 500 字节     |      bytes=0-499       |
  |   表示第二个 500 字节   |     bytes=500-999      |
  |    表示最后 500 字节    |       bytes=-500       |
  | 表示 500 字节以后的范围 |       bytes=500-       |
  |  第一个和最后一个字节   |     bytes=0-0，-1      |
  |    同时指定几个范围     | bytes=500-600，601-999 |

- `UA_Pixels`、`UA-Color`、`UA-OS`、`UA-CPU`：由某些版本的 IE 浏览器所发送的非标准的请求头，表示屏幕大小、颜色深度、操作系统和 CPU 类型

#### 3.3 请求体

以文本或者其他形式组织的请求数据。若方法字段是 GET，则请求体为空时表示没有请求体数据；若请求字段是 POST，则一般此处放置的是要提交的数据。

在请求体前面还有一个空行，用来告诉服务器请求头到此为止。

### 4.响应报文

HTTP 的响应报文也由 3 部分组成：响应行 + 响应头 + 响应体，具体如下：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302250052233.png" alt="image-20230225005245214" style="zoom: 33%;" />

#### 4.1 响应行

一般由协议版本、状态码及其描述组成，例如“HTTP/1.1 200 OK”。

常见的状态码如下所示

|  状态码  |                             说明                             |
| :------: | :----------------------------------------------------------: |
| 100～199 | 表示成功接受请求，要求客户端继续提交下一次请求才能完成整个处理 |
| 200～299 |          表示成功接受请求并完成了整个处理，常用 200          |
| 300～399 |               未完成请求，客户需进一步细化请求               |
| 400～499 | 客户端的请求有错误，常用 404（请求的资源在 Web 服务器中没有），403（服务器拒绝访问，如权限不够） |
| 500～599 |                   服务端出现错误，常用 500                   |

> **说明**
>
> 响应码 401（Unauthorized） 和 403（Forbidden） 都是拒绝访问的意思，区别如下：
>
> - 401 表示服务端不知道客户端是谁。例如 Token 失效、缺失、伪造等会导致服务端无法表示客户端的身份，这时会返回 401，客户端只能重试
> - 403 表示服务端已经知道了客户端是谁，但是客户端没有权限去访问该数据资源

#### 4.2 响应头

用于描述服务器的基本信息和数据描述，服务器通过这些的描述信息可以通知客户端如何处理等一会回送的数据。例如可以用来完成设置 Cookie、指定修改日期、指示浏览器按照指定的间隔刷新页面、声明文档的长度`Content_length`以便利用持有 HTTP 连接等许多其他任务。

设置 HTTP 响应头时往往和状态码结合起来。例如 401 状态代码必须伴随一个`WWW-authenticate`头部来表示未授权。

常见的响应头字段大致如下：

- `Allow`：服务器支持哪些请求方法
- `Content-Encoding`：文档的编码类型，如 gzip 压缩格式。客户端只有在解码后才能得到`Content-Type`头指定的内容类型。由于服务端返回 gzip 压缩文档能够显著地减少 HTML 文档的下载时间，因此服务端应该通过查看`Accept-Encoding`请求头检查客户端是否支持 gzip，为支持 gzip 的客户端返回经 gzip 压缩的 HTML 页面，而为不支持 gzip 的其他客户端返回普通页面
- `Content-Length`：表示内容长度，只有当客户端使用持久 HTTP 连接时才需要这个数据
- `Content-Type`：表示后面的文档属于什么 MIME 类型。Servlet 程序默认为 text/plain，但通常需要显示的指定为 text/html。由于经常要设置`Content-Type`，Sevlet 程序可以调用`HttpServletResponse`提供的一个专用`setContentType()`方法设置该值
- `Date`：当前的 GMT 时间，例如“Date:Mon, 31Dec200104:25:57GMT”。Date 描述的时间表示世界标准时，换算成本地时间，需要知道用户所在的时区。可以用`setDateHeader()`来设置 Date，以避免转换时间格式的麻烦
- `Expires`：告诉客户端需要把回送的资源缓存多长时间，-1 或 0 表示不缓存
- `Last-Modified`：文档的最后改动时间。和客户端请求头配合使用，客户端可以通过请求头`If-Modified-Since`提供一个起始时间，只有改动时间迟于指定起始时间的文档才会返回，否则返回一个 304（Not Modified）状态
- `Location`：配合 302 状态码使用，用于重定向接受者到一个新 URI 地址，表示客户应当到哪里去提取重定向文档
- `Refresh`：告诉客户端隔多久刷新一次，单位秒
- `Server`：服务器通过这个头告诉客户端服务器的类型，例如原始服务器的软件信息等
- `Set-Cookie`：设置和页面关联的 Cookie
- `Transfer-Encoding`：告诉客户端数据的传送格式
- `WWW-Authenticate`：告诉客户端应该在`Authorization`请求头中提供什么类型的授权信息，如果响应状态码为 401，则这个头是必需的

#### 4.3 响应体

可以是文本内容或者二进制内容，比如 JSON、HTML 等都属于纯文本内容。

### 5.GET和POST区别

1. **请求数据的放置位置不同**

   - 对于 GET 请求，请求的数据将会附在 URL 之后，例如`www.test.com/get?name=zhangsan&password=123`

     如果数据是英文字母和数字，就会原样发送；如果数据是特殊字符中的空格，则转移为`+`；如果是中文或者其他字符，则把字符串用 BASE64 加密

   - 对于 POST 请求，提交的数据将被放置在 HTTP 请求报文的请求体中

2. **传输数据的大小不同**

   HTTP 没有对传输的数据大小进行限制，协议规范也没有对 URL 长度进行限制，但是在实际开发中是存在限制的：

   - 对于 GET 请求，特定浏览器和服务器对 URL 长度有限制，例如 IE 对 URL 长度的限制是 2083 字节。对于其他浏览器，如 Netscape、FireFox 等，理论上没有长度限制，其限制取决于操作系统的支持
   - 对于 POST 请求，不是通过 URL 传值的，理论上数据不受限。而实际上各个 Web 服务器会通过自定义设置对 POST 提交数据大小进行限制，Tomcat、Apache、IIS6 都有各自的配置

3. **传输数据的安全性不同**

   这里的安全性并不是指传输过程中的数据安全，仅仅指数据可见性维度的浅层次数据安全。由于 GET 提交数据用户名和密码将明文出现在 URL 上，其安全性比 POST 略低一些。

## 8.3 HTTP的演进

HTTP 在 1.1 版本之前具有无状态的特点，每次请求都需要通过 TCP 三次握手四次挥手与服务器重新建立连接。比如某个客户端在短时间多次请求同一个资源，服务器并不能区别是否已经响应过用户的请求，所以每次都需要重新响应请求。为了节省资源消耗，HTTP 引入了持久连接的方法来进行连接复用。

|   版本   | 产生时间 |                             内容                             |     发展现状      |
| :------: | :------: | :----------------------------------------------------------: | :---------------: |
| HTTP/0.9 |   1991   | 不涉及数据包传输，规定客户端和服务器之间的通信格式，只能 GET 请求 | 没有作为正式标准  |
| HTTP/1.0 |   1996   |  传输内容格式不限制，增加 PUT、PATCH、HEAD、OPTIONS、DELETE  |   正式作为标准    |
| HTTP/1.1 |   1997   | 持久连接（长连接）、节约带宽、HOST 域、管道机制、分块传输编码 | 2015 年前广泛使用 |
| HTTP/2.0 |   2015   |       多路复用、服务器推送、头部信息压缩、二进制协议等       |   逐渐覆盖市场    |

#### 1.HTTP的1.0版本

第一个版本是 0.9 版本，只允许客户端发送 GET 请求，且不支持请求头，因此它只支持纯文本。不过网页仍然支持用 HTML 语言格式化，同时无法插入图片。

第二个版本是 1.0 版本，也是第一个在通信中指定版本号的 HTTP 版本。相对于 HTTP 0.9 版本，1.0 版本中增加了如下特性：

- 请求与响应支持头部字段
- 响应对象以一个响应状态行开始
- 响应对象不只限于超文本
- 支持客户端通过 POST 方法向 Web 服务器提交数据，支持 GET、HEAD、POST 方法
- 支持长连接，但默认使用短连接，缓存机制，以及身份认证
- 请求行必须在尾部添加协议版本字段（HTTP 1.0），并且必须包含头部信息
- 支持 Cache，当客户端在规定时间内访问同一 URL 资源时，直接访问 Cache 即可

**`Content-Type`头部**

因为支持了请求头，请求访问的资源不再局限于 HTML 格式，客户端使用`Content_Type`表示具体请求中的媒体类型，服务端可以使用`Content_Type`来表示具体响应体中的媒体类型，媒体类型（MediaType）全称是互联网媒体类型（Internet MediaType），也叫做 MIME（多用途互联网邮件扩展）类型。

常见的`Content-Type`如下所示：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302250147307.png" alt="image-20230225014727290" style="zoom: 25%;" />

MIME 类型的每个值包括一级类型和二级类型，用`/`分隔。也可以自定义类型，例如`application/vnd.debian.binary-package`，表明发送的是 Debian 系统的二进制数据包。

MIME 类型值还可以在尾部使用分号、添加参数，例如`Content-Type: text/html; charset=utf-8`，表明 HTTP 报文中的内容是文本网页数据，并且文本的编码是 UTF-8.

客户端在发送请求时可以使用 Accept 头部字段声明自己可以接受那些数据格式，例如`Accept: */*`，表明客户端声明自己可以接受来自服务端的任何格式的数据。

**`Content-Encoding`头部**

HTTP 1.0 版本协议支持把数据压缩后再发送，`Content-Encoding`头部字段用于说明数据的压缩格式，具体如下：

|      头部字段的压缩格式      |                   说明                    |
| :--------------------------: | :---------------------------------------: |
| `Content-Encoding: deflate`  | 使用 RFC1950 说明的 zlib 格式进行数据压缩 |
|   `Content-Encoding: gzip`   | 使用 RFC1952 说明的 gzip 格式进行数据压缩 |
| `Content-Encoding: compress` |  使用 UNIX 的文件压缩程序对数据进行压缩   |

客户端在请求时可以使用`Accept-Encoding`字段说明自己可以接受那些压缩方式，例如`Accept-Encoding: gzip,deflate`，表明客户端可以接受 zlib、gzip 格式的压缩数据。

除此之外，HTTP 1.0 还引入了响应状态码、多字符集支持、多部分发送（Multi-Part Type）、权限等。

**TCP 连接**

默认情况下 HTTP 1.0 版本中每次发送一个请求需要一个 TCP 连接，当服务器响应后就会关闭这次连接，下一个请求需再次建立 TCP 连接，这点和 HTTP 0.9 的处理方式一致。

TCP 连接的新建成本很高，因为建立连接时客户端和服务器三次握手，并且连接建立之初数据的发送速率很慢。所以 HTTP 1.0 版本和 HTTP 0.9 版本传输性能都比较差。为了解决这个问题，有些浏览器在请求时增加一个一个非标准的`Connection`头部字段，如果要对传输层的 HTTP 连接进行复用，需要设置`Connection: keep-alive`。这个头部字段要求服务器不要关闭 TCP 连接，以便其他 HTTP 请求复用，同样服务器要回应相同的头部。

如果连接的两端都有`Connection: keep-alive`头部，则会建立一个可以复用的 TCP 连接，直到客户端或服务器主动关闭连接。但是`Connection`不是标准字段，不同服务端实现的行为可能不一致，因此不是提高传输性能的最终解决办法。

#### 2.HTTP的1.1版本

目前主流的版本，在这个版本引入了许多关键技术进行传输性能的优化，如持久连接、管道机制、分块传输编码、字节范围请求等。

**持久连接**

最大的变化就是引入了持久连接，即下层的 TCP 连接默认不关闭，可以被多个请求复用，而且报文不用声明`Connection: keep-alive`头部值。在 HTTP 1.1 版本中，默认一个 TCP 连接可以允许多个 HTTP 请求。

由于客户端和服务端都可以进行通信检测，如果发现对方在一段时间没有活动，就可以主动关闭 TCP 连接。不过相对规范的做法是在客户端最后一个请求时发送`Collection: close`请求头的 HTTP 报文，要求服务器关闭 TCP 连接。

对于同一个域名（带端口），大多数浏览器允许同时建立 6 个持久连接，在降低传输延迟的同时也提高了带宽的利用率。

**管道机制**

在同一个 TCP 连接里允许多个请求同时发送，增加了并发性，进一步改善了 HTTP 协议的效率。举例来说，客户端需要请求两个资源：以前的做法是在同一个 TCP 连接里先发送 A 请求，然后等待服务器做出响应，收到后再发出 B 请求；管道机制则允许浏览器同时发出 A 请求和 B 请求，但是服务器还是按照顺序先响应 A 请求，完成后再响应 B 请求。

**请求头部**

HTTP 1.1 版本新增了 PUT、PATCH、OPTION、DELETE 等多种请求方法。

在请求头部还新增了`Host`字段，用来指定服务器的域名。在 HTTP 1.0 版本中，协议认为每台服务器都绑定一个唯一的 IP 地址，因此请求消息中的 URL 并没有传递主机名。随着虚拟主机技术的发展，在一台物理服务器上可以存在多个虚拟主机（MultiHomed Web Servers），并且他们共享一个 IP 地址甚至是端口号，为虚拟主机的兴起打下了基础。

有了`Host`字段，就可以将请求发往同一台服务器上的不同网站，也可以实现在一台 Web 服务器上的同一组 IP 地址和端口号上使用不同的主机名来创建多个虚拟 Web 站点，或者说，多个虚拟 Server 可以共享同一组 IP 地址和端口号。如果在 HTTP 1.1 版本的请求消息中没有`Host`头部字段，很多服务器会报告 400 错误。

**状态码**

HTTP 1.1 版本新增了 24 个错误状态响应码，如 409（Conflict）表示请求的资源与资源的当前状态发生冲突；410（Gone）表示服务器上的某个资源被永久性地删除。

HTTP 1.1 版本还新加入了一个状态码 100（Continue），服务端通过该响应码告知客户端继续发送后面的请求。例如，客户端事先发送一个只带令牌的`Authorization`头部字段而不带 BOdy 的请求，如果服务器因为权限拒绝了请求就响应状态码 401；如果服务器通过权限校验而验收次请求，就响应状态码 100，客户端就可以继续发送带实体的完整请求了。

使用新的状态码 100 可以允许客户端在发送具有较大 Body 体积的消息之前用 Request Header 试探一下 Server，看 Server 要不要接受 Body，再决定是不是发 Body。当 Body 的体积比较大时，在验证不能通过的情况下能大大节约带宽，传输的性能优势很大。

**缓存**

HTTP 1.1 版本加入了一些缓存的新特性。当缓存对象的 Age 超过 Expire 时，缓存对象变为 Stale 对象之后，HTTP 1.0 版本会直接抛弃 Stale 对象，HTTP 1.1 版本可以不直接抛弃 Cache 中的 Stale 对象，而是与源服务器进行重新验证操作。

**字节范围请求**

HTTP 1.1 版本支持传送内容的一部分，当客户端已经拥有请求资源的一部分后，只需与服务器请求另外的部分资源即可。“字节范围请求”是支持文件断点续传的基础。

具体来说，“字节范围请求”是通过`Range`头部实现的，HTTP 1.0 版本每次传送文件都只能从文件头（0 字节处）开始，在 HTTP 1.1 版本中，客户端通过`Range: bytes=XX`的请求头部值表示要求服务器从文件的“XX”字节处开始传送，即断点续传。其对应的部分内容的响应码是 206（Partial Content）。

**分块传输编码**

分块传输编码（Chunked Transfer Encoding，CTE）是一种新数据传输机制，允许服务端将数据分成多个部分发送到客户端。普通的服务端响应会将响应数据的长度通过`Content-Length`字段告诉客户端。

不过使用`Content-Length`的前提是，服务器发送回应之前，必须知道回应的数据长度。对于一些很耗时的动态操作来说，这意味着服务器要等到所有操作完成才能发送数据，效率不高。更好的处理方法是，产生一块数据就发送一块，采用“流模式”发送取代“缓存模式”发送。

因此，HTTP 1.1 版本规定请求或者响应报文可以不使用`Content-Length`字段，而使用分段传输编码字段，只要请求或回应的头部有`Transfer-Encoding`字段，就表明数据将由数量未定的数据块组成。例如`Transfer-Encoding: chunked`。

每个分块报文的非空的数据块之前会有一个 16 进制的数据，表示当前块的长度。最后是一个大小为 0 的块，表示本次响应的数据发完了。

分块传输编码的具体传输规则为：

- 在头部加入`Transfer-Encoding: chunked`之后，就代表这个报文采用了分块编码。这时报文中的实体需要改为用一系列分块来传输
- 每个分块包含 16 进制的长度值和数据，其中长度值独占一行；长度不包括分块长度后面结尾`CRLF(\r\n)`的长度，也不包括分块数据后面结尾的`CRLF`长度
- 最后一个分块的长度值必须为 0，对应的分块数据没有内容，表示所有的 Body 数据传输完成

例：

```bash
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked


25
THis is the data in the first chunk


1c
and this is the second one


3
con


8
sequence


0
```

> **注意**
>
> 示例中的 25、1C、3、8、0 为 16 进制的分片内容的净长度，并且不包括分片内容后面的`\r\n`的长度。

因为 HTTP 1.1 采用了持久的连接，许多请求（或响应）分片在一个 TCP 的连接上发送，只有第一个分片报文有 HTTP 头部，并且最后通过一个长度为 0 的分片表示当前的 Body 结束。

#### 3.HTTP的2.0版本

HTTP/2.0 协议引入了新的通信单位：帧、消息、流。服务器单位时间接收到的请求数变多了，并发数提高了，并且为多路复用提供了底层支持。

HTTP 的 2.0 版本是一个二进制协议。二进制更易于 Frame（帧、数据包）的传输。HTTP 1.x 版本在应用层以纯文本的形式进行通信，2.0 版本将所有的传输信息分割成更小的消息的数据帧，并对它们采用**二进制格式编码**。为了保证 HTTP 的各种方法、首部不受影响，有需要通过二进制进行传输，因此它在应用层和传输层之间增加了一个二进制分桢层，在该层上 HTTP 2.0 版本会将所有传输的信息分割为更小的消息和数据帧，并对它们采用二进制格式的编码。

HTTP/2.0 协议有 10 个不同 Frame 定义，其中两个最为基础的 Frame 是 Data 帧和 Headers 帧，其中 HTTP/1.x 报文的头部信息会被封装为 HTTP/2.0 报文的 Headers 帧中，而 HTTP/1.x 报文的请求体则被封装到 HTTP/2.0 报文的 Data 帧中。

HTTP 2.0 最大的特点是没有改动 HTTP 的语义，包括 HTTP 方法、状态码、URI 及请求头首部字段等，只是在应用层使用二进制分帧方式传输，实现了低延迟和高吞吐量。其他主要特点有首部压缩、多路复用、并行双向传输、服务端推送等。

**首部压缩**

HTTP/2.0 协议在客户端和服务端使用“**首部（请求头）表**”来跟踪和存储之前发送的请求头键值对，对于相同的数据，不再通过每次请求和响应发送；通信期间几乎不会改变通用键值对的值，所以请求头只需发送一次即可。如果请求中不包含首部，那么首部开销就是零字节。此时所有首部都自动使用之前请求发送的首部。

如果请求的首部发生了变化，只需要在 Headers 帧里发送变化了的首部，将新增或修改的首部帧追加到“首部表”即可。首部表中的键值对在 HTTP/2.0 协议的 TCP 连接存续期间内始终存在，由客户端和服务器共同渐进地更新。

**多路复用**

HTTP/2.0 协议的多路复用指对多资源的请求可以在一个 TCP 连接上完成。HTTP/2.0 协议把 HTTP 协议通信的基本单位缩小为一个一个的帧，这些帧对应着逻辑流中的消息，并行地在同一个 TCP 连接上双向交换消息。

HTTP 性能的关键在于低延迟而不是带宽利用率低。大多数 HTTP 连接的时间都很短，数据传输是突发性的，但是 TCP 传输只有在长连接并且传输大块数据时效率才是最高的。HTTP/2.0 协议通过让所有数据流共用同一个连接，可以更有效地让 TCP 连接高带宽，也能真正地服务于 HTTP 的性能提升。

特点如下：

- 可以减少服务连接压力，内存占用少，连接吞吐量大
- 由于 TCP 连接减少而改善了网络拥塞状况
- TCP 慢启动时间减少，拥塞和丢包恢复速度快

**并行双向传输**

在 HTTP/2.0 协议中，客户端和服务器可以把 HTTP 消息分解为互不依赖的帧，然后乱需发送，最后在另一端把它们重新组合起来。因为同一连接上可以有多个不同方向的数据流在传输，客户端可以一边乱序发送消息流，一边接收服务器的响应流，服务器同理。

特点如下：

- 可以并行交错地发送请求，请求间互不影响
- 可以并行交错地发送响应，响应间互不影响
- 只使用一个连接即可并行发送多个请求和响应
- 消除不必要的延迟，减少页面加载时间

**服务端推送**

HTTP/2.0 协议中服务器可以对一个客户端请求发送多个响应，而无需客户端明确请求。

## 8.4 案例：基于Netty实现简单的Web服务器

Netty 是异步事件驱动的架构，相比于传统的 Tomcat、Jetty 等 Web 容器，具有轻量小巧的特点，更适合作为 Web 服务器使用。

本节实现一个简单的 HTTP 回显服务器 HttpEchoServer，将 HTTP 请求的请求方法、请求参数、请求 URI、请求头、请求体等内容进行回显。其服务端的 Pipeline 大致如下：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302252123000.png" alt="image-20230225212349845" style="zoom:50%;" />

### 1.基于Netty的HTTP请求处理流程

Netty 内置的 HTTP 请求的编解码处理器：

- `HttpRequestDecoder`：HTTP 请求解码器，入站处理器，间接地继承了`ByteToMessageDecoder`，将 ByteBuf 缓冲区解码成代表请求的 HttpRequest 首部实例和 HttpContent 内容实例，并且`HttpRequestDecoder`在解码时会处理好分块（Chunked）类型和固定长度（Content-Length）类型的 HTTP 请求报文
- `HttpResponseEncoder`：HTTP 响应编码器，出站处理器，把`HttpResponse`首部实例和 HttpContent 内容实例编码成 ByteBuf 字节流
- `HttpServerCodec`：HTTP 的编解码器，是`HttpRequestDecoder`解码器和`HttpResponseEncoder`编码器的结合体
- `HttpObjectAggregator`：是`HttpObject`实例聚合器，也是一个入站处理器。通过`HttpObject`实例聚合器，可以把`HttpMessage`首部实例和一个或多个`HttpContent`内容实例最终聚合成一个`FullHttpRequest`实例。`HttpMessage`、`HttpRequest`、`HttpContent`、`FullHttpRequest`等类型都是`HttpObject`的子类
- `QueryStringDecoder`：把 HTTP 的请求 URI 分割成 Path 路径和 Key-Value 参数，同一次请求该解码器仅能使用一次

基于 Netty 的 HTTP 请求的处理流程大致如下：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302252147599.png" alt="image-20230225214701582" style="zoom:50%;" />

- 二进制的 HTTP 数据包从 Channel 通道入站后，首先进入 Pipeline 流水线的是 ByteBuf 字节流
- `HttpRequestDecoder`将 ByteBuf 缓冲区的请求行和请求头解析成`HttpRequest`首部对象，传入到`HttpObjectAggregator`，然后将 HTTP 数据包的请求体解析成`HttpContent`对象（可能多个），传入到`HttpObjectAggregator`，解码完成后，如果没有更多的请求体内容，`HttpRequestDecoder`会传递一个`LastHttpContent`结束实例到聚合器`HttpObjectAggregator`，表示 HTTP 请求数据解析完成

- 当`HttpObjectAggregator`收到`LastHttpContent`实例，就会将收到的全部`HttpObject`实例封装成一个`FullHttpRequest`整体请求实例发送给下一站

在请求体处理过程中会涉及`Content-Length`和`Trunked`两种类型请求体，但是其处理差异被`HttpRequestDecoder`协议解码器所屏蔽，它们的最终出站对象是一致的，通过聚合器`HttpObjectAggregator`处理之后输出的都是`FullHttpRequest`实例。HTTP 服务端的业务处理器可以通过该`FullHttpRequest`实例获取到所有与 HTTP 请求的内容。

装配处理器的代码大致如下：

```java
ChannelPipeline pipeline = ch.pipeline();
pipeline.addLast(new HttpRequestDecoder());
pipeline.addLast(new HttpObjectAggregator(65535));
pipeline.addLast(new HttpResponseEncoder());
pipeline.addLast(new HttpEchoHandler());
```

### 2.Netty内置的HTTP报文解码流程

最终，通过`HttpRequestDecoder`和`HttpObjectAggregator`对 HTTP 请求报文进行处理后，Netty 会将 HTTP 请求封装为一个`FullHttpRequest`实例。

Netty 内置的与 HTTP 请求报文相对应的类大致有如下几个：

- `FullHttpRequest`：包含整个 HTTP 请求的信息，组合了`HttpRequest`首部和`HttpContent`请求体
- `HttpRequest`：请求首部，主要包含对 HTTP 请求行和请求头的组合
- `HttpContent`：对 HTT 请求体进行封装，本质上是一个 ByteBuf 缓冲区实例。如果 ByteBUf 的长度固定，则请求体过大，可能包含多个`HttpContent`；解码时，最后一个解码返回对象为`LastHttpContent`（空的`HttpContent`），表示请求体的解码结束
- `HttpMethod`：对 HTTP 请求方法的封装
- `HttpVersion`：对 HTTP 版本的封装，定义了 HTTP/1.0 和 HTTP/1.1 两个版本协议
- `HttpHeaders`：封装了 HTTP 报文请求头

各个部分的对应关系如下：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302252204873.png" alt="image-20230225220417846" style="zoom:50%;" />

Netty 的`HttpRequest`首部类中有一个 String uri 成员，主要是对请求 URI 的封装，包含了 Path 路径和请求参数。

对于请求参数的解析，不同的 Web 服务器使用的解析策略不同。在 Tomcat 中，如果客户端提交的是`application/x-www-form-urlencoded`类型的表单 POST 请求，则 Java 请求参数实例除了包含跟随在 URI 后面的键值对外，请求参数还包含 HTTP 请求体 Body 中的键值对。在 Netty 中，Java 中请求参数实例仅仅包含跟在 URI 后面的键值对。

**Netty 的 HTTP 报文拆包方案**

一般来说，服务端收到的 HTTP 字节流可能被分成多个 ByteBuf 包，Netty 大致有如下策略来处理分包问题：

1. 定长分包策略：接收端按照固定长度进行数据包分割，发送端按照固定长度进行发送

2. 长度域分包策略：比如使用`LengthFieldBasedFrameDecoder`长度域解码器在接收端分包，则发送端先发送消息的长度字段，在发送消息的内容

3. `分隔符分割`：比如使用`LineBasedFrameDecoder`解码器通过换行符进行分包，或者使用`DelimiterBasedFrameDecoder`通过特定的分隔符进行分包

Netty 结合了第 2 种和第 3 种策略完成 HTTP 报文的拆包：对于请求头，应用分隔符分包策略，以特定分隔符`\r\n`进行拆包；对于 HTTP 请求体，应用长度字段中的分包策略，按照请求头中的内容长度进行内容拆包。

Netty 总体的 HTTP 拆包方案如下：

- 处理 HTTP 请求行，由于 Header 的边界是 CRLF，如果读取到 CRLF，则意味着请求行的信息已经读取完成
- 处理请求头部分，由于 Header 的边界是 CRLF，每遇到一个 CRLF，表示一个请求头读取完成；如果连续读到两个 CRLF，则表示全部 Header 的信息读取完成
- 请求体的长度一般由请求头`Content-Length`来进行确定；如果没有`Content-Length`头部，则属于“块编码”报文，具体的解析方式按照 Trunked 协议

为了介绍内存复制，Netty 使用了`CompositeByteBuf`。例如，Netty 聚合各个`HttpObject`实例的`FullHttpMessage`实现类就是一个`CompositeByteBuf`实例，该组合缓冲区会将`HttpRequest`内部的 ByteBuf 、`HttpContent`内部的 ByteBuf 都组合在一起，作为最终的 HTTP 报文缓冲区，从而避免数据拷贝。

### 3.基于Netty的HTTP响应编码流程

Netty 的 HTTP 相应的处理流程只需在流水线装配`HttpResponseEncoder`编码器即可，它具有以下特点：

- 输入的是`FullHttpResponse`响应实例，输出的是 ByteBuf 字节缓冲期。后面的处理器会将 ByteBuf 数据写入 Channel，最终发送到 HTTP 客户端
- 该编码器按照 HTTP 对入站`FullHttpResponse`实例的请求行、请求头、请求体进行序列化，通过请求头去判断是否含有`Content-Length`头或者`Trunked`头，然后将请求体按照相应的长度规则对内容进行序列化

如果只是发送简单的 HTTP 响应，可以通过`DefaultFullHttpResponse`默认响应实现类完成。通过该默认响应类既可以设置响应的内容，也可以进行响应头的设置。

使用的代码如下：

```java
public class HttpProtocolHelper{
    
	public static void sendJsonContent(ChannelHandlerContext ctx, String content){
        HttpVersion version = getHttpVersion(ctx);
         // 构造一个默认的 FullHttpResponse 实例
        FullHttpResponse response = new DefaultFullHttpResponse(version, OK, Unpooled.copiedBuffer(content, CharsetUtil.UTF_8));
         // 设置响应头
        response.headers().set(CONTENT_TYPE, "application/json; charset=UTF-8");
         // 发送响应内容
        sendAndCleanupConnection(ctx, response);
    } 
    
    private static HttpVersion getHttpVersion(ChannelHandlerContext ctx){
        HttpVersion version;
        if (isHTTP_1_0(ctx))
            version = HttpVersion.HTTP_1_0;
        else
            version = HttpVersion.HTTP_1_1;
        return version;
    }
    
    public static boolean isHTTP_1_0(ChannelHandlerContext ctx){
        HttpVersion protocol_version = ctx.channel().attr(PROTOCOL_VERSION_KEY).get();
        if (null == protocol_version)
            return false;
        if (protocol_version.equals(HttpVersion.HTTP_1_0))
            return true;
        return false;
    }
    
    public static void sendAndCleanupConnection(ChannelHandlerContext ctx, FullHttpResponse response){
        final boolean keepAlive = ctx.channel().attr(KEEP_ALIVE_KEY).get();
        HttpUtil.setContentLength(response, response.content().readableBytes());
        if (!keepAlive){
            // 如果不是长连接，设置 connection:close 头部
            response.headers().set(CONNECTION, CLOSE);
        } else if (isHTTP_1_0(ctx)){
            // 如果是 1.0 版本的长连接，设置 connection:keep-alive 头部
            response.headers().set(CONNECTION, KEEP_ALIVE);
        }
        // 发送内容
        ChannelFuture writePromise = ctx.channel().writeAndFlush(response);
        if (!keepAlive)
            // 如果不是长连接，发送完成之后，关闭连接
            writePromise.addListener(ChannelFutureListener.CLOSE);
    }
}
```

### 4.回显业务处理器

将来自客户端的 HTTP 请求方法、URI 请求参数、请求体数据、请求头字段回显到客户端。

大致代码如下：

```java
public class HttpEchoHandler extends SimpleChannelInboundHandler<FullHttpRequest>{

    @Override
    public void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) throws Exception{
        if (!request.decoderResult().isSuccess()){
            HttpProtocolHelper.sendError(ctx, BAD_REQUEST);
            return;
        }
        /**
         * 缓存 HTTP 协议的版本号
         */
        HttpProtocolHelper.cacheHttpProtocol(ctx, request);

        Map<String, Object> echo = new HashMap<String, Object>();
        // 1.获取 URI
        String uri = request.uri();
        echo.put("request uri", uri);
        // 2.获取请求方法
        HttpMethod method = request.method();
        echo.put("request method", method.toString());
        // 3.获取请求头
        Map<String, Object> echoHeaders = new HashMap<String, Object>();
        HttpHeaders headers = request.headers();
        Iterator<Map.Entry<String, String>> hit = headers.entries().iterator();
        while (hit.hasNext()){
            Map.Entry<String, String> header = hit.next();
            echoHeaders.put(header.getKey(), header.getValue());
        }
        echo.put("request header", echoHeaders);
        /**
         * 获取 uri 请求参数
         */
        Map<String, Object> uriDatas = paramsFromUri(request);
        echo.put("paramsFromUri", uriDatas);
        // 处理 POST 请求
        if (POST.equals(request.method())){
            /**
             * 获取请求体数据到 map
             */
            Map<String, Object> postData = dataFromPost(request);
            echo.put("dataFromPost", postData);
        }

        /**
         * 回显内容转换成 json 字符串
         */
        String sendContent = JsonUtil.pojoToJson(echo);
        /**
         * 发送回显内容到客户端
         */
        HttpProtocolHelper.sendJsonContent(ctx, sendContent);

    }

    /*
     * 从 URI 后面获取请求的参数
     */
    private Map<String, Object> paramsFromUri(FullHttpRequest fullHttpRequest)
    {
        Map<String, Object> params = new HashMap<String, Object>();
        // 调用 Netty 自带方法把 URI 后面的参数串分割成 key-value 形式
        QueryStringDecoder decoder = new QueryStringDecoder(fullHttpRequest.uri());
        // 提取 key-value 形式的参数串
        Map<String, List<String>> paramList = decoder.parameters();
        // 迭代 key-value 形式的参数串
        for (Map.Entry<String, List<String>> entry : paramList.entrySet()){
            params.put(entry.getKey(), entry.getValue().get(0));
        }
        return params;
    }

    /*
     * 获取 POST 方式传递的请求体数据
     */
    private Map<String, Object> dataFromPost(FullHttpRequest fullHttpRequest){
        Map<String, Object> postData = null;
        try{
            String contentType = fullHttpRequest.headers().get("Content-Type").trim();
            // 普通 form 表单数据，非 multipart 形式表单
            if (contentType.contains("application/x-www-form-urlencoded")){
                postData = formBodyDecode(fullHttpRequest);
            }
            // multipart 形式表单
            else if (contentType.contains("multipart/form-data")){
                postData = formBodyDecode(fullHttpRequest);
            }
            // 解析 json 数据
            else if (contentType.contains("application/json")){
                postData = jsonBodyDecode(fullHttpRequest);
            } else if (contentType.contains("text/plain")){
                ByteBuf content = fullHttpRequest.content();
                byte[] reqContent = new byte[content.readableBytes()];
                content.readBytes(reqContent);
                String text = new String(reqContent, "UTF-8");
                postData = new HashMap<String, Object>();
                postData.put("text", text);
            }
            return postData;
        } catch (UnsupportedEncodingException e){
            return null;
        }
    }

    /*
     * 解析 from 表单数据
     */
    private Map<String, Object> formBodyDecode(FullHttpRequest fullHttpRequest){
        Map<String, Object> params = new HashMap<String, Object>();
        try{
            HttpPostRequestDecoder decoder = new HttpPostRequestDecoder(new DefaultHttpDataFactory(DefaultHttpDataFactory.MINSIZE),
                            fullHttpRequest,
                            CharsetUtil.UTF_8);
            List<InterfaceHttpData> postData = decoder.getBodyHttpDatas();
            if(postData==null || postData.isEmpty()){
                 decoder = new HttpPostRequestDecoder(fullHttpRequest);
                if(fullHttpRequest.content().isReadable()){
                    String json=fullHttpRequest.content().toString(CharsetUtil.UTF_8);
                    params.put("body", json);
                }
            }

            for (InterfaceHttpData data : postData){
                if (data.getHttpDataType() == InterfaceHttpData.HttpDataType.Attribute){
                    MixedAttribute attribute = (MixedAttribute) data;
                    params.put(attribute.getName(), attribute.getValue());
                }
            }
        } catch (IOException e){
            e.printStackTrace();
        }
        return params;
    }

    /*
     * 解析 json 数据（Content-Type = application/json）
     */
    private Map<String, Object> jsonBodyDecode(FullHttpRequest fullHttpRequest) throws UnsupportedEncodingException{
        Map<String, Object> params = new HashMap<String, Object>();
        ByteBuf content = fullHttpRequest.content();
        byte[] reqContent = new byte[content.readableBytes()];
        content.readBytes(reqContent);
        String strContent = new String(reqContent, "UTF-8");
        JSONObject jsonParams = JsonUtil.jsonToPojo(strContent, JSONObject.class);
        for (Object key : jsonParams.keySet()){
            params.put(key.toString(), jsonParams.get(key));
        }
        return params;
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause){
        cause.printStackTrace();
        if (ctx.channel().isActive()){
            HttpProtocolHelper.sendError(ctx, INTERNAL_SERVER_ERROR);
        }
    }

}
```

### 5.发送请求

**发送`application/x-www/form-urlencoded`编码类型**

该类型的请求体会将表单的每个表单项名称和值转换为“名称=值”的形式，然后用“&”连在一起，最终将整个表单编码后的字符串作为 POST 请求的请求体发送出去。如果是 GET 请求，则将编码后的字符串追加到 URI 后面发送出去。

拦截 POST 请求的应用层 HTTP 协议数据包，结果如下：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302252253605.png" alt="image-20230225225350573" style="zoom:50%;" />

**发送`multipart/form-data`编码类型**

浏览器在编码`multipart/form-data`类型时，会将每一个表单项分开进行编码。每个表单项都有一个`Content-disposition`来说明表单项的类型，表单 Field 字段的类型值为 form-data（数据）、File 字段的类型值为 file（文件）。紧跟在`Content-disposition`属性后面，每一个表单项都有一个 name 属性，其值为表单项的名称。在 name 之后是两个`\r\n`，然后是表单项的值，如果上传文件，则此处为文件的内容；每个表单项的末尾都有一段`boundary`分隔字符串，隔开自己和下一个表单项。

示例

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302252256509.png" alt="image-20230225225653478" style="zoom: 25%;" />

编码之后的 Form 表单被同一个`boundary`分隔符分开，`boundary`分隔符的值则被包含在请求的`Content-Type`请求头的后半部分，处于“multipart/form-data”的后面，如下图所示：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302252258503.png" alt="image-20230225225859472" style="zoom: 25%;" />
