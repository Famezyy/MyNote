# 第10章_WebSocket原理与实战

WebSocket 是一种全双工通信的协议，其通信在 TCP 连接上进行，所以属于应用层协议。WebSocket 使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在 WebSocket 编程中，浏览器和服务器只需要完成一次升级握手就可以直接创建持久性的连接，并进行双向数据传输。

对于 WebSocket 的 Java 开发，Java 官方发布了 JSR-356 规范，该规范的全称为 Java API for WebSocket。不少 Web 容器（如 Tomcat、Jetty 等）都支持 JSR-356 规范，提供了 WebSocket 应用开发 API。Tomcat 从 7.0.27 开始支持 WebSocket，从 7.0.47 开始支持 JSR-356 规范。

无论是 Tomcat 还是 Jetty，其性能在高并发场景下的表现并不是非常理想。所以编写 WebSocket 服务端程序时一般基于 Netty 框架进行编写。

> **扩展**
>
> 一个具备在线聊天、在线推送功能的综合性 WebSocket 项目：https://gitee.com/crazymaker/websocket_chat_room

## 1.WebSocket协议简介

WebSocket 协议的目标是在一个独立的持久连接上提供全双工双向通信。客户端和服务端可以向对方主动发送和接收数据。WebSocket 通信协议于 2011 年被 IETF 发布为 RFC6455 标准，后又发布了 RFC7936 标准补充规范。WebSocket API 也被 W3C（World Wid Web Consortium，万维网联盟）定为标准。

### 1.1 Ajax短轮训和Long Poll长轮训的原理

在 WebSocket 双向通信技术前，浏览器与服务器之间的双向通信大致有两种方式：Ajax 短轮询和 Long Poll 长轮询。

#### 1.Ajax短轮询

Ajax 短轮询即浏览器周期性地向服务器发起 HTTP 请求，不管服务器是否真正获取到数据，都会向浏览器返回响应，浏览器通过 HTTP 1.1 的持久连接（建立一次 TCP 连接，发送多个请求），可以在建立一次 TCP 连接之后发起多个异步请求。



