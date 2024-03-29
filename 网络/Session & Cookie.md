---
typora-copy-images-to: ./img/Session & Cookie
---

## 1.Cookie

在了解这三个概念之前我们先要了解HTTP是**无状态**的 Web 服务器，什么是无状态呢？就像上面一次对话完成后下一次对话完全不知道上一次对话发生了什么。如果在 Web 服务器中只是用来管理静态文件还好说，对方是谁并不重要，把文件从磁盘中读取出来发出去即可。但是随着网络的不断发展，比如电商中的购物车只有记住了用户的身份才能够执行接下来的一系列动作。所以此时就需要我们**无状态**的服务器记住一些事情。

那么 Web 服务器是如何记住一些事情呢？既然 Web 服务器记不住东西，那么我们就在外部想办法记住，相当于服务器给每个客户端都贴上了一个小纸条。上面记录了服务器给我们返回的一些信息。然后服务器看到这张小纸条就知道我们是谁了。那么 `Cookie` 是谁产生的呢？Cookies 是由服务器产生的。接下来我们描述一下 `Cookie` 产生的过程：

- 浏览器第一次访问服务端时，服务器此时肯定不知道他的身份，所以创建一个独特的身份标识数据，格式为 `key=value`，放入到 `Set-Cookie` 字段里，随着响应报文发给浏览器
- 浏览器看到有 `Set-Cookie` 字段以后就知道这是服务器给的身份标识，于是就保存起来，下次请求时会自动将此 `key=value` 值放入到 `Cookie` 字段中发给服务端
- 服务端收到请求报文后，发现 `Cookie` 字段中有值，就能根据此值识别用户的身份然后提供个性化的服务

<img src="img/Session & Cookie/202309261454565.png" alt="image-20220515171754148" style="zoom: 50%;" />

接下来我们用代码演示一下服务器是如何生成，我们自己搭建一个后台服务器，这里我用的是 SpringBoot 搭建的，并且写入SpringMVC 的代码如下。

```java
@RequestMapping("/testCookies")
public String cookies(HttpServletResponse response){
    response.addCookie(new Cookie("testUser","xxxx"));
    return "cookies";
}
```

项目启动以后我们输入路径 `http://localhost:8005/testCookies`，然后查看发的请求。可以看到下面那张图使我们首次访问服务器时发送的请求，可以看到服务器返回的响应中有 `Set-Cookie` 字段。而里面的 `key=value` 值正是我们服务器中设置的值。

<img src="img/Session & Cookie/202309261454567.png" alt="image-20220515171906231" style="zoom: 80%;" />

接下来我们再次刷新这个页面可以看到在请求体中已经设置了 `Cookie` 字段，并且将我们的值也带过去了。这样服务器就能够根据 `Cookie` 中的值记住我们的信息了。

<img src="img/Session & Cookie/202309261454568.png" alt="image-20220515171928069" style="zoom: 80%;" />

接下来我们换一个请求呢？是不是 `Cookie` 也会带过去呢？接下来我们输入路径 `http://localhost:8005` 请求。我们可以看到 `Cookie` 字段还是被带过去了。

<img src="img/Session & Cookie/202309261454569.png" alt="image-20220515171947568" style="zoom: 50%;" />

浏览器的 `Cookie` 存放位置可以按照下面步骤查看：

1. 在计算机打开 `Chrome`
2. 在右上角，一次点击“更多”图标->“设置”
3. 在底部，点击“高级”
4. 在“隐私设置和安全性”下方，点击网站设置
5. 依次点击 “Cookie” -> 查看所有“Cookie和网站数据”

然后可以根据域名进行搜索所管理的 `Cookie` 数据。所以是浏览器替你管理了 `Cookie` 的数据，如果此时你换成了 `Firefox` 等其他的浏览器，因为 `Cookie` 刚才是存储在 `Chrome` 里面的，所以服务器又蒙圈了，不知道你是谁，就会给 `Firefox` 再次贴上小纸条。

<img src="img/Session & Cookie/202309261454570.png" alt="image-20220515172033012" style="zoom:67%;" />

### 1.1 Cookie中的参数设置 

说到这里，应该知道了 `Cookie` 就是服务器委托浏览器存储在客户端里的一些数据，而这些数据通常都会**记录用户的关键识别信息**。所以 `Cookie` 需要用一些其他的手段用来保护，防止外泄或者窃取，这些手段就是 `Cookie` 的属性。

| 参数名   | 作用                                                         | 后端设置方法               |
| -------- | ------------------------------------------------------------ | -------------------------- |
| Max-Age  | 设置 cookie 的过期时间，单位为秒                             | `cookie.setMaxAge(10)`     |
| Domain   | 指定了 Cookie 所属的域名                                     | `cookie.setDomain("")`     |
| Path     | 指定了 Cookie 所属的路径                                     | `cookie.setPath("");`      |
| HttpOnly | 告诉浏览器此 Cookie 只能靠浏览器 Http 协议传输，禁止其他方式访问 | `cookie.setHttpOnly(true)` |
| Secure   | 告诉浏览器此 Cookie 只能在 Https 安全协议中传输，如果是 Http 则禁止传输 | `cookie.setSecure(true)`   |

下面我就简单演示一下这几个参数的用法及现象。

#### 1.Path

设置为 `cookie.setPath("/testCookies")`，接下来我们访问 `http://localhost:8005/testCookies`，我们可以看到在左边和我们指定的路径是一样的，所以`Cookie`才在请求头中出现，接下来我们访问 `http://localhost:8005`，我们发现没有 `Cookie` 字段了，这就是 `Path` 控制的路径。

<img src="img/Session & Cookie/202309261454571.png" alt="image-20220515172210345" style="zoom:80%;" />

#### 2.Domain

设置为 `cookie.setDomain("localhost")`，接下来我们访问 `http://localhost:8005/testCookies` 我们发现下图中左边的是有 `Cookie` 的字段的，但是我们访问 `http://172.16.42.81:8005/testCookies`，看下图的右边可以看到没有 `Cookie` 的字段了。这就是 `Domain` 控制的域名发送 `Cookie`。

<img src="img/Session & Cookie/202309261454572.png" alt="image-20220515172239443" style="zoom:80%;" />

## 2.Session

### 2.1 简介

> **注意**
>
> `Cookie` 是存储在**客户端**方，用作传输 `SessionId` 的媒介，`Session` 是存储在**服务端**方，底层是一个 `ConcurrentMap`，用来识别每个用户，客户端只存储 `SessionId`。**禁止存储对象**！最好只用于存储登陆认证相关信息。

如果将账户的一些信息都存入 `Cookie` 中的话，一旦信息被拦截，那么我们所有的账户信息都会丢失掉。所以就出现了 `Session`，在一次会话中将重要信息保存在 `Session` 中，浏览器只记录 `SessionId` 一个 `SessionId` 对应一次会话请求。

<img src="img/Session & Cookie/202309261454573.png" alt="image-20220603195550541" style="zoom:80%;" />

```java
@RequestMapping("/testSession") 
@ResponseBody 
public String testSession(HttpSession session){ 
    session.setAttribute("testSession","this is my session"); 
    return "testSession"; 
} 


@RequestMapping("/testGetSession")
@ResponseBody
public String testGetSession(HttpSession session){
    Object testSession = session.getAttribute("testSession");
    return String.valueOf(testSession);
}
```

这里我们写一个新的方法来测试 `Session` 是如何产生的，我们在请求参数中加上 `HttpSession session`，然后再浏览器中输入 `http://localhost:8005/testSession` 进行访问可以看到在服务器的返回头中在 `Cookie` 中生成了一个 `SessionId`。然后浏览器记住此 `SessionId` 下次访问时可以带着此 Id，然后就能根据此 Id 找到存储在服务端的信息了。

<img src="img/Session & Cookie/202309261454574.png" alt="image-20220515172519071" style="zoom: 50%;" />

此时我们访问路径 `http://localhost:8005/testGetSession`，发现得到了我们上面存储在 `Session` 中的信息。

<img src="img/Session & Cookie/202309261454575.png" alt="image-20220515172602476" style="zoom:80%;" />

`Session` 只有在后端调用相关方法时才会创建，与生命周期相关的方法有：`getCreationTime()`、`getLastAccessedTime()`、`setMaxInactiveInternal()`、`getMaxInactiveInterval()` 等方法。

客户端的 `Session` 生命周期和 `Cookie` 一致，如果没设置过期时间，默认是关了浏览器就没了，即再打开浏览器的时候初次请求头中是没有 `SessionId` 了；服务端在 `LastAccessedTime` 之后再过 `MaxInactiveInterval` 时间后 `Session` 就会过期，默认 30 分钟。

### 2.2 Tomcat源码

`Session` 的管理是在容器中被管理的，接下来我们拿最常用的 `Tomcat` 为例来看下 `Tomcat` 是如何管理 `Session` 的。在 `ManageBase` 的 `createSession方法` 是用来创建 `Session` 的。

```java
@Override
public Session createSession(String sessionId) {
    //首先判断 Session 数量是不是到了最大值，最大 Session 数可以通过参数设置
    if ((maxActiveSessions >= 0) &&
        (getActiveSessions() >= maxActiveSessions)) {
        rejectedSessions++;
        throw new TooManyActiveSessionsException(
            sm.getString("managerBase.createSession.ise"),
            maxActiveSessions);
    }

    // 重用或者创建一个新的 Session 对象，注意在 Tomcat 中就是 StandardSession
    // 它是 HttpSession 的具体实现类，而 HttpSession 是 Servlet 规范中定义的接口
    Session session = createEmptySession();

    // 初始化新 Session 的值
    session.setNew(true);
    session.setValid(true);
    session.setCreationTime(System.currentTimeMillis());
    // 设置 Session 过期时间是 30 分钟
    session.setMaxInactiveInterval(getContext().getSessionTimeout() * 60);
    String id = sessionId;
    if (id == null) {
        id = generateSessionId();
    }
    session.setId(id);// 这里会将 Session 添加到 ConcurrentHashMap 中
    sessionCounter++;

    //将创建时间添加到 LinkedList 中，并且把最先添加的时间移除
    //主要还是方便清理过期 Session
    SessionTiming timing = new SessionTiming(session.getCreationTime(), 0);
    synchronized (sessionCreationTiming) {
        sessionCreationTiming.add(timing);
        sessionCreationTiming.poll();
    }
    return session;
}
```

创建出来后 `Session` 会被保存到一个 `ConcurrentHashMap` 中，可以看 `StandardSession` 类。

```java
protected Map<String, Session> sessions = new ConcurrentHashMap<>();
```

> **注意**
>
> `Session` 是存储在 Tomcat 的容器中，所以如果后端机器是多台的话，因此多个机器间是无法共享 `Session` 的，此时可以使用 Spring 提供的分布式 `Session` 的解决方案，原理是将 `Session` 放在了 Redis 中。

## 3.Token

`Session` 是将要验证的信息存储在服务端，并以 `SessionId` 和数据进行对应，`SessionId` 由客户端存储，在请求时将 `SessionId` 也带过去，因此实现了状态的对应。而 `Token` 是在服务端将用户信息经过 Base64Url 编码过后传给在客户端，每次用户请求的时候都会带上这一段信息，因此服务端拿到此信息进行解密后就知道此用户是谁了，这个方法叫做 `JWT(Json Web Token)`。

<img src="img/Session & Cookie/202309261454576.png" alt="image-20220515173023151" style="zoom: 80%;" />

### 3.1 Token的优点 

1. 简洁：可以通过 `URL`、`POST` 参数或者是在 `HTTP` 头参数发送，因为数据量小，传输速度也很快
2. 自包含：由于串包含了用户所需要的信息，避免了多次查询数据库
3. 因为 Token 是以 Json 的形式保存在客户端的，所以 JWT 是跨语言的
4. 不需要在服务端保存会话信息，特别适用于分布式微服务

### 3.2 JWT令牌

JWT（JSON Web Token）是一种用户凭证的编码规范，是一种网络环境下编码用户凭证的 JSON 格式的开放标准（RFC 7519）。

<img src="img/Session & Cookie/202309261454577.png" alt="image-20220515173238210" style="zoom:67%;" />

一个编码后的 JWT 令牌字符串分为三部分：header + payload + signature。这三部分通过 `.` 连接。

#### 1.Header

是一个 Json 对象，描述 JWT 的元数据，通常是下面这样子的

```json
{
    "alg": "HS256",
    "typ": "JWT"
}
```

上面代码中，`alg` 属性表示签名的算法（algorithm），默认是 HMAC SHA256（写成 HS256）；`typ` 属性表示这个令牌的类型（type）。

最后，将上面的 JSON 对象使用 Base64URL 算法转成字符串。

> **补充**
>
> JWT 作为一个令牌（token），有些场合可能会放到 URL（比如 api.example.com/?token=xxx）。Base64 有三个字符 `+`、`/` 和 `=`，在 URL 里面有特殊含义，所以要被替换掉：
>
> = 被省略、+ 替换成 -，/ 替换成 _ 
>
> 这就是 Base64URL 算法。

#### 2.Payload

Payload 部分也是一个 Json 对象，用来存放实际需要传输的数据，JWT 规定了下面几个官方的字段供选用。

- `iss` (issuer)：签发人
- `exp` (expiration time)：过期时间
- `sub` (subject)：主题
- `aud` (audience)：接收 jwt 的一方
- `nbf` (Not Before)：定义在什么时间之前，该 jwt 都是不可用的
- `iat` (Issued At)：签发时间
- `jti` (JWT ID)：jwt 的唯一身份标识，用来作为一次性 token，保持幂等性

当然除了官方提供的这几个字段我们也能够自己定义私有字段，下面就是一个例子

```json
{
    "name": "xiaoMing",
    "age": 14
}
```

默认情况下 JWT 是不加密的，任何人只要在网上进行 Base64 解码就可以读到信息，所以一般不要将秘密信息放在这个部分。这个 Json 对象也要用 `Base64URL` 算法转成字符串。

#### 3.Signature

JWT 的第三部分是一个签名字符串，这一部分是将 header 的 Base64 编码和 payload 的 Base64 编码使用点号 `.` 连接起来后，通过 header 声明的加密算法进行加密所得到的密文。为了保证安全，加密时需要加入盐。

首先需要定义一个秘钥，这个秘钥只有服务器才知道，不能泄露给用户，然后使用 Header 中指定的签名算法（默认情况是HMAC SHA256），算出签名以后将 Header、Payload、Signature 三部分拼成一个字符串，每个部分用 `.` 分割开来，就可以返给用户了。

服务端会验证 token，如果验证通过就会从中获取信息，可以保证数据的完整性和防止伪造。

> HS256 可以使用单个密钥为给定的数据样本创建签名。当消息与签名一起传输时，接收方可以使用相同的密钥来验证签名是否与消息匹配。

### 3.3 Java中的使用

上面我们介绍了关于 JWT 的一些概念，接下来如何使用呢？首先在项目中引入 Jar 包：

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.4</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.4</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.4</version>
</dependency>
```

然后编码如下

```java
public class JWTUtil {

    private static final SecretKey SECRET_KEY = Jwts.SIG.HS256.key().build();

    // 过期时间，1 小时
    private static final long expirationTime = 60 * 60 * 1000;

    public static String generateJwtToken(Map<String, Object> claims) {
        long now = System.currentTimeMillis();
        return Jwts.builder()
                .issuer("youyi.zhao")
                .subject("security")
                .issuedAt(new Date(now))
                .expiration(new Date(now + expirationTime))
                .claims(claims)
                .signWith(SECRET_KEY, Jwts.SIG.HS256)
                .compact();
    }

    // 如果 token 过期则会自动抛出 ExpiredJwtException
    public static Claims getPayload(String token) throws ExpiredJwtException {
            return  Jwts.parser()
                    .verifyWith(SECRET_KEY)
                    .build()
                    .parseSignedClaims(token)
                    .getPayload();
    }

    public String refreshToken(String token) {
        Claims payload = getPayload(token);
        return generateJwtToken(payload);
    }

}
```

发现输出的 Token 如下

```java
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJzdWJqZWN0IiwiaXNzIjoiaXNzdWVyIiwibmFtZSI6InhpYW9NaW5nIiwiYWdlIjoxNH0.3KOWQ-oYvBSzslW5vgB1D-JpCwS-HkWGyWdXCP5l3Ko
```

此时在网上随便找个 Base64 解码的网站就能将 header 和 payload 的信息解码出来

```bash
eyJhbGciOiJIUzI1NiJ9 - {"alg":"HS256"}
eyJzdWIiOiJzdWJqZWN0IiwiaXNzIjoiaXNzdWVyIiwibmFtZSI6InhpYW9NaW5nIiwiYWdlIjoxNH0 - {"sub":"subject","iss":"issuer","name":"xiaoMing","age":14}
```

## 4.Cookie相对Token的优势

**（1）无状态**

基于 token 的验证是无状态的，这也许是它相对 cookie 来说最大的优点。后端服务不需要记录 token。每个令牌都是独立的，包括检查其有效性所需的所有数据，并通过声明传达用户信息。

服务器唯一的工作就是在成功的登陆请求上签署 token，并验证传入的 token 是否有效。

**（2）防跨站请求伪造（CSRF）**

假设在网页中有这样的一个链接：`![](http://bank.com?withdraw=1000&to=tom)`，假设你已经通过银行的验证并且 cookie 中存在验证信息，同时银行网站没有 CSRF 保护。一旦用户点了这个图片，就很有可能从银行向 tom 这个人转 1000 块钱。

但是如果银行网站使用了 token 作为验证手段，攻击者将无法通过上面的链接转走你的钱。（因为攻击者无法获取正确的 token）

**（3）多站点使用**

cookie 绑定到单个域。foo.com 域产生的 cookie 无法被 bar.com 域读取。使用 token 就没有这样的问题。这对于需要向多个服务获取授权的单页面应用程序尤其有用。

使用 token，使得用从 myapp.com 获取的授权向 myservice1.com 和 myservice2.com 获取服务成为可能。

**（4）支持移动平台**

好的 API 可以同时支持浏览器，iOS 和 Android 等移动平台。然而，在移动平台上，cookie 是不被支持的。

**（5）性能**

一次网络往返时间（通过数据库查询 session 信息）总比做一次 HMACSHA256 计算的 Token 验证和解析要费时得多。