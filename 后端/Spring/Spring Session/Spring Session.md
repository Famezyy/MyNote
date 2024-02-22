# Spring Session

[官网](https://spring.io/projects/spring-session)

## 1.简介

在分布式微服务架构下，需要在服务节点之间进行会话的共享。解决方案是使用一个统一的 Session 数据库来保存会话数据并实现共享，这样的数据库应该是轻量级的基于内存的高速数据库。

在生产环境中，可以使用成熟稳定的 Spring Session 开源组件作为分布式 Session 的解决方案。Spring Session 作为独立的组件将 Session 从 Web 容器中剥离，存储在独立的数据库中，目前支持多种形式的数据库：内存数据库、关系型数据库、文档行数据库（如 MogonDB ）等。

## 2.核心组件

这里介绍 Spring Session 的 3 个核心组件：`Session` 接口、`RedisSession` 会话类、`SessionRepository` 存储接口。

### 2.1 Session接口

该接口是 Spring Session 对会话的抽象，主要是为了鉴定用户，为 HTTP 请求和响应提供上下文容器。

主要方法如下：

- `getId`：获取 Session ID
- `setAttribute`：设置会话属性
- `getAttribute`：获取会话属性
- `setLastAccessedTime`：设置会话过程中最近的访问时间
- `getLastAccessedTime`：获取最近的访问时间
- `setMaxInactiveInterValInSeconds`：设置会话的最大闲置时间
- `getMaxInactiveInterValInSeconds`：获取最大闲置时间
- `isExpired`：判断会话是否过期

> **注意**
>
> Spring Session 和 Tomcat 的 Session 在实现模式上有很大不同，Tomcat 中直接实现 Servlet 规范的 `HttpSession` 接口，而 Spring Session 中则抽象出单独的 `Session` 接口，用于应对不同的传输、存储场景。同时 Spring Session 定义了一个适配器类，可以将 `Session` 实例适配成 Servlet 规范中的 `HttpSession` 实例。

### 2.2 RedisSession会话类

用于使用 Redis 进行会话属性存储的场景，它有两个非常重要的成员属性：

- `cached`

  它是一个 `MapSession` 实例，用于进行本地缓存，每次在进行 `getAttribute` 操作时优先从本地缓存获取，没有取到再从 Redis 中获取，以提升性能。而 `MapSession` 是由 Spring Security Core 定义的一个通过内部的 `HashMap` 缓存键-值对的本地缓存类。

- `delta`

  用于跟踪变化数据，目的是保存变化的 `Session` 属性。`RedisSession` 提供了 `saveDelta` 方法，用于持久化 Session 到 Redis 中。

### 2.3 SessionRepository存储接口

管理 Spring Session 的存储接口，主要方法如下：

- `createSession`：创建 Session 实例
- `findById`：根据 id 查找 Session 实例
- `delete`：根据 id 删除 Session 实例
- `save`：存储 Session 实例

根据 Session 的实现类不同，Session 存储实现类分为多种：`RedisSession` 会话的存储类为 `RedisOperationsSessionRepository`，其负责 Session 数据到 Redis 数据库的读写。

`RedisSession` 在 Redis 缓存中的存储细节大致有 3 种 Key：

```bash
spring:session:SESSION_KEY:sessions:0cefe354-3c24-40d8-a859-fe7d9d3c0dba
spring:session:SESSION_KEY:expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe
spring:session:SESSION_KEY:expirations:1581695640000
```

第一种 Key 用来存储 Session 的详细信息，Key 的最后部分为 Session ID，这是一个 UUID。这个 Key 的 Value 在 Redis 中是一个 hash 类型，内容包括 Session 的过期时间间隔、最近的访问时间、属性等。Key 的过期时间为 Session 的最大过期时间 +5 分钟。如果设置的 Session 过期时间为 30 分钟，则这个 Key 的过期时间为。35 分钟。

第二种 Key 用来表示 Session 在 Redis 中已经过期，这个键值对不存储任何有用数据，只是为了表示 Session 过期而设置。

第三种 Key 存储过去一段时间内过期的 Session ID 集合。这个 Key 的最后部分是一个时间戳，代表计时的起始时间。这个 Key 的 Value 所使用的 Redis 数据结构是 set，set 中的元素是时间戳滚动至下一分钟计算得出的过期 Session Key（第二种）。

## 3.使用

首先需要导入依赖：

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

配置

```properties
spring.data.redis.host=43.153.170.51
spring.data.redis.port=30379
# 指定 cookie 的存放路径为根路径
# server.servlet.session.cookie.path=/
# 指定 cookie 的存放域名
# server.servlet.session.cookie.domain=
```

然后调用 `httpSession` 的相关方法就会发现 session 存储在了 redis 中。

```java
@RestController
public class MyTestController {

    @GetMapping("/get")
    public String get(HttpSession httpSession) {
        return httpSession.getAttribute("hello").toString();
    }

    @GetMapping("/set")
    public String set(HttpSession httpSession) {
        httpSession.setAttribute("hello", "world");
        return "ok";
    }
}
```

## 4.整合Spring Security

```java
@Configuration
public class SecurityConfiguration<S extends Session> {

	// 注入一个 sessionRepository
	@Autowired
	private FindByIndexNameSessionRepository<S> sessionRepository;

	@Bean
	SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
		return http
			// other config goes here...
			.sessionManagement((sessionManagement) -> sessionManagement
				.maximumSessions(2)
                // 指定 sessionRepository
				.sessionRegistry(sessionRegistry())
			)
			.build();
	}

	@Bean
	public SpringSessionBackedSessionRegistry<S> sessionRegistry() {
		return new SpringSessionBackedSessionRegistry<>(this.sessionRepository);
	}

}
```

