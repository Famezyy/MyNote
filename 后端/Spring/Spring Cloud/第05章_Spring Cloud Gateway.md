# 第05章_Spring Cloud Gateway

所谓的 API 网关，就是指系统的统一入口，它封装了应用程序的内部结构，为客户端提供统一服务，一些与业务本身功能无关的公共逻辑可以在这里实现，诸如认证、鉴权、监控、路由转发等等。

<img src="img/image-20231009200639200.png" alt="image-20231009200639200" style="zoom:80%;" />

## 1.简介

Spring Cloud Gateway 是 Spring Cloud 官方推出的响应式的 API 网关，定位于取代 Netflix Zuul1.0。相比 Zuul 来说，Spring Cloud Gateway 提供更优秀的性能，更强大的有功能。它不能在传统的 servlet 容器中工作，也不能构建成 war 包。Spring Cloud Gateway 旨在为微服务架构提供一种简单且有效的 API 路由的管理方式，并基于 Filter 的方式提供网关的基本功能，例如安全认证、监控、限流等等。

### 1.1 Spring Cloud Gateway 功能特征

- 基于Spring Framework 5, Project Reactor 和 Spring Boot 2.0 进行构建
- 动态路由：能够匹配任何请求属性
- 支持路径重写
- 集成 Spring Cloud 服务发现功能（Nacos、Eruka）
- 可集成流控降级功能（Sentinel、Hystrix）
- 可以对路由指定易于编写的 Predicate（断言）和 Filter（过滤器）

### 1.2 核心概念

- 路由（route) 

  路由是网关中最基础的部分，路由信息包括一个ID、一个目的URI、一组断言工厂、一组Filter组成。如果断言为真，则说明请求的URL和 配置的路由匹配。

- 断言(predicates) 

  Java8 中的断言函数，SpringCloud Gateway 中的断言函数类型是 Spring5.0 框架中的 ServerWebExchange。断言函数允许开发者去定义匹配 Http request 中的任何信息，比如请求头和参数等。

- 过滤器（Filter) 

  SpringCloud Gateway 中的 filter 分为 Gateway FilIer 和 Global Filter。Filter 可以对请求和响应进行处理。

### 1.3 工作原理

<img src="https://raw.githubusercontent.com/Famezyy/picture/cab167bef2a44ad40e32632600b7c8d8c6b0eb0a/notePictureBed/image-20231009201248443.png" alt="image-20231009201248443" style="zoom: 67%;" />

## 2.快速开始

新建项目

1. **引入依赖**

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>
       
       <parent>
           <groupId>com.youyi.zhao</groupId>
           <artifactId>springcloudalibaba</artifactId>
           <version>0.0.1-SNAPSHOT</version>
       </parent>
       
       <groupId>com.example</groupId>
       <artifactId>gate-way</artifactId>
       <version>0.0.1-SNAPSHOT</version>
       <name>gate-way</name>
       <description>gate-way</description>
       
       <dependencies>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-gateway</artifactId>
           </dependency>
       </dependencies>
       
   </project>
   ```

   > **注意**
   >
   > `gateway`依赖会和`spring-webmvc`的依赖冲突，需要在父 POM 中将`spring-boot-starter-web`放入`dependencyManagement`中：
   >
   > 1. 将`spring-boot-starter-web`放到`DependencyManagement`中，并声明 version 和`spring-boot-starter-parent`一致
   >
   >    ```xml
   >    <dependency>
   >        <groupId>org.springframework.boot</groupId>
   >        <artifactId>spring-boot-starter-web</artifactId>
   >        <version>${spring-boot-start-web.version}</version>
   >    </dependency>
   >    ```
   >
   > 2. 在其他微服务中添加`spring-boot-starter-web`
   >
   > 或者在配置文件中添加`spring.main.web-application-type=reactive`（推荐）。

2. **配置文件**

   ```yaml
   server:
   	port: 8080
   spring:
       main:
       	web-application-type: reactive
       application:
   		name: api‐gateway
       cloud:
           gateway:
               routes: # 路由数组（路由就是指定当请求满足什么条件的时候转到哪个微服务）
                 ‐ id: order_route # 当前路由的标识，要求唯一
                   uri: http://localhost:8081 # 请求要转发到的地址
                   order: 1 # 路由的优先级,数字越小级别越高
                   predicates: # 断言(就是路由转发要满足的条件)
                     ‐ Path=/order-serv/** # 当请求路径满足 Path 指定的规则时才进行路由转发
                   filters: # 过滤器，请求在传递过程中可以通过过滤器对其进行一定的修改
                     ‐ StripPrefix=1 # 转发之前去掉 1 层路径
   ```

3. **测试**

   当访问`http://localhost:8080/order-serv/order/add`时会路由到`http://localhost:8081/order/add`。

## 3.集成Nacos

### 3.1 集成注册中心

1. **引入依赖**

   ```xml
   <dependencies>
       <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-gateway</artifactId>
       </dependency>
       <dependency>
           <groupId>com.alibaba.cloud</groupId>
           <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
       </dependency>
       <!-- 配合 lb 使用  -->
       <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-loadbalancer</artifactId>
       </dependency>
   </dependencies>
   ```

2. **编写 yml 配置文件**

   ```yaml
   server:
     port: 8080
   
   spring:
     main:
       web-application-type: reactive
     application:
       name: api‐gateway
     cloud:
       nacos:
         discovery:
           server‐addr: 192.168.11.100:8848
           username: nacos
           password: nacos
       gateway:
         routes:
           - id: order_route
             uri: lb://order-server # lb 表示从 nacos 中寻找服务，遵循负载均衡策略，需要导入 LoadBalancer 依赖！
             predicates:
               - Path=/order-serv/**
             filters:
               - StripPrefix=1
   ```

   访问`http://localhost:8080/order-serv/order/add`时会路由到`http://localhost:8081/order/add`。

   **简写方式（不建议）**

   ```yml
   server:
     port: 8080
   
   spring:
     main:
       web-application-type: reactive
     application:
       name: api‐gateway
     cloud:
       gateway:
         discovery:
           locator:
             enabled: true # 启动自动识别 Nacos 服务，根据服务名路由
       nacos:
         discovery:
           server‐addr: 192.168.11.100:8848
           username: nacos
           password: nacos
   ```

   访问`http://localhost:8080/order-server/order/add`时会路由到`http://localhost:8081/order/add`。


### 3.2 集成配置中心

1. **引入依赖**

   ```xml
   <dependencies>
       <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-gateway</artifactId>
       </dependency>
       <dependency>
           <groupId>com.alibaba.cloud</groupId>
           <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
       </dependency>
       <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-loadbalancer</artifactId>
       </dependency>
       <!-- 引入下面两个 config 需要的依赖 -->
       <dependency>
           <groupId>com.alibaba.cloud</groupId>
           <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
       </dependency>
       <!-- 在SpringCloud 2020.* 版本把加载 bootstrap 的功能禁用了，导致在读取文件的时候读取不到而报错，所以我们只要把bootstrap从新导入进来就会生效了 -->
       <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-bootstrap</artifactId>
       </dependency>
   </dependencies>
   ```

2. 移出 springboot.yaml 的配置并新增 bootstrap.yaml 配置

   ```yaml
   spring:
     main:
       web-application-type: reactive
     application:
       name: gateway-config
     cloud:
       nacos:
         server-addr: 192.168.11.100:8848
         username: nacos
         password: nacos
   ```
   
3. 在 NACOS 配置中心新增`Data ID`为 gateway-config 的配置

   ```yaml
   server:
     port: 8080
   
   spring:
     cloud:
       gateway:
         routes:
         - id: after_route
           uri: lb://order-server
           predicates:
           - After=2017-01-20T17:42:47.789-07:00[America/Denver]
       nacos:
         discovery:
           server‐addr: 192.168.11.100:8848
           username: nacos
           password: nacos
   ```

4. 访问即可发现路由配置生效，并且修改配置发布后也能动态感知。

   > **注意**
   >
   > `routes`下新添加规则不支持热更新，只能修改已有规则的断言或者过滤才支持热更新。

## 4.路由断言工厂配置

当请求 gateway 的时候使用断言对请求进行匹配，如果匹配成功就路由转发，如果匹配失败就返回 404，既可以使用内置配置也可以自定义配置。

参考：https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#gateway-request-predicates-factories

### 4.1 内置配置

#### 1.基于Datetime类型的断言工厂

此类型的断言根据时间做判断，主要有三个，日期格式为`ZonedDateTime`的输出格式：

1. `After`：接收一个日期参数，判断请求日期是否晚于指定日期

   ```yaml
   spring:
     cloud:
       gateway:
         routes:
         - id: after_route
           uri: lb://order-server
           predicates:
           - After=2017-01-20T17:42:47.789-07:00[America/Denver]
   ```

2. `Before`：接收一个日期参数，判断请求日期是否早于指定日期

   ```yaml
   spring:
     cloud:
       gateway:
         routes:
         - id: before_route
           uri: lb://order-server
           predicates:
           - Before=2017-01-20T17:42:47.789-07:00[America/Denver]
   ```

3. `Between`：接收两个日期参数，判断请求日期是否在指定时间段内

   ```yaml
   spring:
     cloud:
       gateway:
         routes:
         - id: between_route
           uri: lb://order-server
           predicates:
           - Between=2017-01-20T17:42:47.789-07:00[America/Denver], 2017-01-21T17:42:47.789-07:00[America/Denver]
   ```

#### 2.基于Cookie的断言工厂

接收两个参数，cookie 名字和一个正则表达式，用于判断请求 cookie 是否具有给定名称且值与正则表达式匹配。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: cookie_route
        uri: lb://order-server
        predicates:
        - Cookie=chocolate, ch.p
```

#### 3.基于Header的断言工厂

接收两个参数，标题名称和正则表达式，判断请求 Header 是否具有给定名称且值与正则表达式匹配。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: header_route
        uri: lb://order-server
        predicates:
        - Header=X-Request-Id,\d+
```

#### 4.基于Host的断言工厂

接收一个参数，判断请求的 Host 是否满足匹配规则，多个规则（或关系）用`,`隔开。支持占位符如`{sub}.myhost.org`，该占位符的键值对可以通过在`GatewayFileter`中调用`ServerWebExchange.getAttributes()`获取。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: lb://order-server
        predicates:
        - Host=**.somehost.org,**.anotherhost.org
```

#### 5.基于Method请求方法的断言工厂

接收一个参数，判断请求类型是否跟指定的类型匹配，多个类型（或关系）用`,`隔开

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: lb://order-server
        predicates:
        - Method=GET,POST
```

#### 6.基于Path请求路径的断言工厂

接收一个参数，判断请求的 URI 部分是否满足路径规则，支持多个规则（或关系）和占位符，但是如果`matchTrailingSlash`设置为`false`则不支持占位符。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: path_route
        uri: lb://order-server
        predicates:
        - Path=/red/{segment},/blue/{segment}
```

该占位符的键值对可以通过在`GatewayFileter`中调用`ServerWebExchange.getAttributes()`获取。可以通过`ServerWebExchangeUtils`中的方法快速获取：

```java
Map<String, String> uriVariables = ServerWebExchangeUtils.getUriTemplateVariables(exchange);
String segment = uriVariables.get("segment");
```

#### 7.基于Query请求参数的断言工厂

接收两个参数，请求 param 和正则表达式，判断请求参数是否具有给定名称且值与正则表达式匹配。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: lb://order-server
        predicates:
        # 检查请求参数是否包含 color
        - Query=color
        # 检查请求参数是否包含 color 并且值符合正则表达式
        # -Query=color, gree.
```

#### 8.基于远程地址的断言工厂

接收一个 IP 地址段，判断请求主机地址是否在地址段中。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: remoteaddr_route
        uri: lb://order-server
        predicates:
        - RemoteAddr=192.168.1.1/24
```

#### 9.基于路由权重的断言工厂

接收一个[组名,权重], 然后对于同一个组内的路由按照权重转发。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: weight_high
        uri: https://weighthigh.org
        predicates:
        - Weight=group1, 8
      - id: weight_low
        uri: https://weightlow.org
        predicates:
        - Weight=group1, 2
```

这个规则会转发 80% 的流量到 weighthigh.org 和 20% 的流量到 weightlow.org。

### 4.2 自定义路由断言工厂

自定义路由断言工厂需要继承`AbstractRoutePredicateFactory`类，重写`apply`方法的逻辑。在`apply`方法中可以通过`exchange.getRequest()`拿到`ServerHttpRequest`对象，从而可以获取到请求的参数、请求方式、请求头等信息。可参考其他内置配置类的写法。

书写时必须遵守以下条件：

1. 必须注册为 bean
2. 类必须加上 **RoutePredicateFactory** 作为结尾
3. 必须继承`AbstractRoutePredicateFactory`
4. 必须声明静态内部 model 类，用来接收配置文件中对应的断言的信息
5. 需要使用`shortcutFieldOrder`返回内部类中定义的参数名称
6. 在`apply`中进行逻辑判断，true 就是匹配成功，false 则匹配失败

```java
@Component
public class CheckRoutePredicateFactory extends AbstractRoutePredicateFactory<CheckRoutePredicateFactory.Config> {


    public CheckRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return new Predicate<ServerWebExchange>() {
            @Override
            public boolean test(ServerWebExchange serverWebExchange) {
                return config.getValue1().equals("test1") && config.getValue2().equals("test2");

            }
        };
    }

    @Override
    public List<String> shortcutFieldOrder() {
        // 这里要按照配置文件中的顺序传入所有在 Config 中定义的变量
        return CollectionUtils.list("value1", "value2");
    }

    @Data
    public static class Config {
        private String value1;
        private String value2;
    }
}
```

```yaml
spring:
  main:
    web-application-type: reactive
  application:
    name: api‐gateway
  cloud:
    gateway:
      routes:
        - id: after_route
          uri: lb://order-server
          predicates:
            - Check=test1,test2
```

## 5.过滤器工厂配置

Gateway 内置了很多的过滤器工厂，我们通过一些过滤器工厂可以进行一些业务逻辑处理器，比如添加剔除响应头，添加去除参数等。

参考：https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#gatewayfilter-factories

| 过滤器工厂                  | 作用                                                         | 参数                                                         |
| :-------------------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| AddRequestHeader            | 为原始请求添加 Header                                        | Header 的名称及值                                            |
| AddRequestParameter         | 为原始请求添加请求参数                                       | 参数名称及值                                                 |
| AddResponseHeader           | 为原始响应添加 Header                                        | Header 的名称及值                                            |
| DedupeResponseHeader        | 剔除响应头中重复的值                                         | 需要去重的 Header 名称及去重策略                             |
| Hystrix                     | 为路由引入 Hystrix 的断路器保护                              | HystrixCommand 的名称                                        |
| FallbackHeaders             | 为 fallbackUri 的请求头中添加具体的异常信息                  | Header 的名称                                                |
| PrefixPath                  | 为原始请求路径添加前缀                                       | 前缀路径                                                     |
| PreserveHostHeader          | 为请求添加一个 preserveHostHeader=true 的属性，路由过滤器会检查该属性以决定是否要发送原始的 Host | 无                                                           |
| RequestRateLimiter          | 用于对请求限流，限流算法为令牌桶                             | keyResolver、rateLimiter、statusCode、denyEmptyKey、emptyKeyStatus |
| RedirectTo                  | 将原始请求重定向到指定的 URL                                 | http 状态码及重定向的 url                                    |
| RemoveHopByHopHeadersFilter | 为原始请求删除 IETF 组织规定的一系列 Header                  | 默认就会启用，可以通过配置指定仅删除哪些 Header              |
| RemoveRequestHeader         | 为原始请求删除某个 Header                                    | Header 名称                                                  |
| RemoveResponseHeader        | 为原始响应删除某个 Header                                    | Header 名称                                                  |
| RewritePath                 | 重写原始的请求路径                                           | 原始路径正则表达式以及重写后路径的正则表达式                 |
| RewriteResponseHeader       | 重写原始响应中的某个 Header                                  | Header 名称，值的正则表达式，重写后的值                      |
| SaveSession                 | 在转发请求之前，强制执行`WebSession::save`操作               | 无                                                           |
| SecureHeaders               | 为原始响应添加一系列起安全作用的响应头                       | 无，支持修改这些安全响应头的值                               |
| SetPath                     | 修改原始的请求路径                                           | 修改后的路径                                                 |
| SetResponseHeader           | 修改原始响应中某个 Header 的值                               | Header 名称，修改后的值                                      |
| SetStatus                   | 修改原始响应的状态码                                         | HTTP 状态码，可以是数字，也可以是字符串                      |
| StripPrefix                 | 用于截断原始请求的路径                                       | 使用数字表示要截断的路径的数量                               |
| Retry                       | 针对不同的响应进行重试                                       | retries、statuses、methods、series                           |
| RequestSize                 | 设置允许接收最大请求包的大小。如果请求包大小超过设置的值，则返回`413 Payload Too Large` | 请求包大小，单位为字节，默认值为5M                           |
| ModifyRequestBody           | 在转发请求之前修改原始请求体内容                             | 修改后的请求体内容                                           |
| ModifyResponseBody          | 修改原始响应体的内容                                         | 修改后的响应体内容                                           |
| Default                     | 为所有路由添加过滤器                                         | 过滤器工厂名称及值                                           |

下面介绍几种常用的过滤器。

### 5.1 添加请求头

```yaml
spring:
  cloud:
    gateway:
    # 设置路由：路由 Id、路由到微服务的 uri、断言
      routes:
      ‐ id: order_route # 路由 ID，全局唯一
        uri: http://localhost:8020 # 目标微服务的请求地址和端口
      # 配置过滤器工厂
      filters:
      ‐ AddRequestHeader=X‐Request‐color, red # 添加请求头
```

### 5.2 添加请求参数

```yaml
spring:
  cloud:
    gateway:
      routes:
      ‐ id: order_route
        uri: http://localhost:8020
      filters:
      ‐ AddRequestParameter=color,blue # 添加请求参数
```

### 5.3 为匹配的路由统一添加前缀

```yaml
spring:
  cloud:
    gateway:
      routes:
      ‐ id: order_route
        uri: http://localhost:8020
      filters:
      ‐ PrefixPath=/mall‐order # 添加前缀 对应微服务需要配置 context‐path
```

服务中需要配置：

```yaml
server:
  servlet:
  context‐path: /mall‐order
```

测试：`http://localhost:8888/order/findOrderByUserId/1` ====> `http://localhost:8020/mall­order/order/findOrderByUserId/1`

### 5.4 重定向操作

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: lb://order-server
        predicates:
        - After=2017-01-20T17:42:47.789-07:00[America/Denver]
        filters:
          - RedirectTo=302,https://www.baidu.com
```

测试：`http://localhost:8080/order/add`

### 5.5 自定义过滤器工厂

与自定义路由断言工厂类似，需要继承`AbstractNameValueGatewayFilterFactory`且我们的自定义名称必须要以 **GatewayFilterFactory** 结尾并注册为 bean。

 ```java
 @Component
 public class CheckGatewayFilterFactory extends AbstractGatewayFilterFactory<CheckGatewayFilterFactory.Config> {
 
     public CheckGatewayFilterFactory() {
         super(CheckGatewayFilterFactory.Config.class);
     }
 
     @Override
     public GatewayFilter apply(Config config) {
         return (exchange, chain) -> {
             if (config.getValue() != null) {
                 // 如果没有传入参数 zhao 或者 value 与配置文件不同则返回 404
                 if (!config.getValue().equals(exchange.getRequest().getQueryParams().getFirst("zhao"))) {
                     exchange.getResponse().setStatusCode(HttpStatus.BAD_REQUEST);
                     return exchange.getResponse().setComplete();
                 }
             }
             return chain.filter(exchange);
         };
     }
 
     @Override
     public List<String> shortcutFieldOrder() {
         return Collections.singletonList("value");
     }
 
     @Data
     public static class Config {
         String value;
     }
 }
 ```

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: lb://order-server
        predicates:
        - After=2017-01-20T17:42:47.789-07:00[America/Denver]
        filters:
          - Check=test
```

### 5.6 全局过滤器配置

`GlobalFilter`接口和`GatewayFilter`有一样的接口定义，但是`GlobalFilter`会作用于所有路由。

<img src="img/第05章_Spring Cloud Gateway/image-20231210180105305.png" alt="image-20231210180105305" style="zoom:67%;" />

**自定义全局过滤器**

```java
@Component
public class LogFilter implements GlobalFilter {

    Logger log = LoggerFactory.getLogger(this.getClass());

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info(exchange.getRequest().getPath().value());
        return chain.filter(exchange);
    }
}
```

## 6.Reactor Netty访问日志（了解）

需要在 VM 启动环境变量中添加：`-Dreactor.netty.http.server.accessLogEnabled=true`，开启后会默认打印请求访问日志。

## 7.跨域配置

默认情况下浏览器会阻止跨域请求，例如对于下面的 ajax 请求网页：

```html
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <script src="http://apps.bdimg.com/libs/jquery/1.9.1/jquery.min.js"></script>
</head>
<body>
    <div>
        <table border="1">
            <thead>
              <tr>
                  <th>id</th>
                  <th>username</th>
                  <th>age</th>
              </tr>
            </thead>
            <tbody id="userlist"></tbody>
        </table>
    </div>
    <input type=button value="list" onclick="getData()">
    <script>
        function getData() {
            $.get('http://localhost:8088/order/add', function (data) {
                alert(data)
            })
        }
    </script>
</body>
</html>
```

在 IDEA 中用浏览器打开后运行在 63342 端口，此时由于域名与后端端口不同，如果点击按钮的话会出现如下错误：

<img src="img/第05章_Spring Cloud Gateway/image-20231211195753900.png" alt="image-20231211195753900" style="zoom:67%;" />

**（1）通过 YAML 配置**

https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway/cors-configuration.html

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':                                             # 允许跨域访问的端点
            allowedOrigins: "*"                                # 允许跨域访问的来源，"*"表示所有来源
            allowedMethods:
            - GET
            - POST
```

在配置中心配置好，需要==**重启 Gateway 服务**==，再发起 Ajax 请求发现可以正常得到结果。

**（2）通过 JAVA 配置类配置**

```java
@Configuration
public class CorsConfig {
    @Bean
    public CorsWebFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedMethod("*");
        config.addAllowedOrigin("*");
        config.addAllowedHeader("*");

        // 在网关中添加时需要创建以下的类
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource(new PathPatternParser());
        source.registerCorsConfiguration("/**", config);

        return new CorsWebFilter(source);
    }
}
```

## 8.整合Sentinel流控降级

### 8.1 环境配置

1. 添加依赖

   ```xml
   <!-- spring cloud gateway 核心 -->
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-gateway</artifactId>
   </dependency>
   <!-- 引入sentinel进行服务降级熔断 -->
   <dependency>
       <groupId>com.alibaba.cloud</groupId>
       <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
   </dependency>
   <!-- gateway网关整合sentinel进行限流降级 -->
   <dependency>
       <groupId>com.alibaba.cloud</groupId>
       <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
   </dependency>
   ```

2. properties 文件或 nacos 配置中心中配置 sentinel dashboard 地址

   ```yaml
   spring:
     cloud:
       sentinel:
         eager: true
         transport:
           dashboard: localhost:8080
   ```

3. 添加 VM 启动参数

   ```bash
   -Dcsp.sentinel.app.type=1
   ```

通过 API 网关访问端口后会在控制台生成相应的链路。

### 8.2 配置流控降级规则

spring cloud Gateway 中使用 sentinel 时的降级规则不适用于 4xx 和 5xx 错误。

#### 1.单个资源流控



<img src="img/第05章_Spring Cloud Gateway/image-20231211215550605.png" alt="image-20231211215550605" style="zoom:67%;" />

- 针对请求属性：可以针对 IP、host、header、URL 参数、cookie 来进行流控
- QPS 间隔：表示监测多长时间内 QPS 的阈值

#### 2.API分组流控

也可以新建一个 API 分组统一进行流控：

<img src="img/第05章_Spring Cloud Gateway/image-20231211221844619.png" alt="image-20231211221844619" style="zoom:67%;" />

之后在流控规则中选择 API 分组：

<img src="img/第05章_Spring Cloud Gateway/image-20231211222004899.png" alt="image-20231211222004899" style="zoom:67%;" />

### 8.3 自定义异常

#### 1.通过代码配置

```java
@Configuration
public class GatewayConfiguration {
    @PostConstruct
    public void init() {
        BlockRequestHandler blockRequestHandler = new BlockRequestHandler() {
            @Override
            public Mono<ServerResponse> handleRequest(ServerWebExchange exchange, Throwable ex) {
               Map<String, String> result = new HashMap<>();
                result.put("code", String.valueOf(HttpStatus.TOO_MANY_REQUESTS.value()));
                result.put("msg", HttpStatus.TOO_MANY_REQUESTS.getReasonPhrase());
                return ServerResponse.status(HttpStatus.OK)
                        .contentType(MediaType.APPLICATION_JSON)
                        .body(BodyInserters.fromValue(result));
            }
        };
        GatewayCallbackManager.setBlockHandler(blockRequestHandler);
    }
}
```

#### 2.通过配置文件

可声明在 nacos 中，支持动态更新响应体和响应状态，会覆盖代码配置。

```yaml
spring:
  cloud:
    sentinel:
      #配置限流之后的响应内容
      scg:  
        fallback:
          # 两种模式：一种是response返回文字提示信息，一种是redirect，重定向跳转，需要同时配置redirect（跳转的 uri）
          mode: response
          # 响应的状态
          response-status: 426
          # 响应体
          response-body: '{"code": 426,"message": "限流了，稍后重试！"}'
```

### 8.4 代码方式加载网关规则（了解）

```java
@PostConstruct
public void doInit() {
    
    // 初始化自定义的 API
    initCustomizedApis();
    // 初始化网关规则
    initGatewayRules();
    // 设置自定义异常处理器
    initBlockRequestHandler();
}
    
private void initCustomizedApis() {
    Set<ApiDefinition> definitions = new HashSet<>();
    ApiDefinition api1 = new ApiDefinition("customized_api")
            .setPredicateItems(new HashSet<ApiPredicateItem>() {{
                add(new ApiPathPredicateItem().setPattern("/api/**")
                        .setMatchStrategy(SentinelGatewayConstants.URL_MATCH_STRATEGY_PREFIX));
            }});

    ApiDefinition api2 = new ApiDefinition("book_content_api")
            .setPredicateItems(new HashSet<ApiPredicateItem>() {{
                add(new ApiPathPredicateItem().setPattern("/api/book/queryBookContent**")
                        .setMatchStrategy(SentinelGatewayConstants.URL_MATCH_STRATEGY_PREFIX));
            }});
    definitions.add(api1);
    definitions.add(api2);
    GatewayApiDefinitionManager.loadApiDefinitions(definitions);

}

/**
 * 自定义网关限流规则
 * 1.对所有api接口通过IP进行限流,每个IP，2秒钟内请求数量大于10，即视为爬虫
 * 2.对小说内容接口访问进行限流，每个IP，1秒钟请求数量大于1，则视为爬虫
 * */

private void initGatewayRules() {
    Set<GatewayFlowRule> rules = new HashSet<>();
    // resource：资源名称，可以是网关中的 route 名称或者自定义的 API 分组
    // count：限流阈值
    // intervalSec：统计时间窗口，默认 1 秒
    rules.add(new GatewayFlowRule("customized_api")
            .setResourceMode(SentinelGatewayConstants.RESOURCE_MODE_CUSTOM_API_NAME)
            .setCount(10)
            .setIntervalSec(2)
            .setParamItem(new GatewayParamFlowItem()
                    .setParseStrategy(SentinelGatewayConstants.PARAM_PARSE_STRATEGY_CLIENT_IP)
            )
    );

    rules.add(new GatewayFlowRule("book_content_api")
            .setResourceMode(SentinelGatewayConstants.RESOURCE_MODE_CUSTOM_API_NAME)
            .setCount(1)
            .setIntervalSec(1)
            .setParamItem(new GatewayParamFlowItem()
                    .setParseStrategy(SentinelGatewayConstants.PARAM_PARSE_STRATEGY_CLIENT_IP)
            )
    );
    // 加载网关规则
    GatewayRuleManager.loadRules(rules);
}

private void initBlockRequestHandler() {
    BlockRequestHandler blockRequestHandler = new BlockRequestHandler() {
        @Override
        public Mono<ServerResponse> handleRequest(ServerWebExchange exchange, Throwable ex) {
            Map<String, String> result = new HashMap<>();
                result.put("code", String.valueOf(HttpStatus.TOO_MANY_REQUESTS.value()));
                result.put("msg", HttpStatus.TOO_MANY_REQUESTS.getReasonPhrase());
                return ServerResponse.status(HttpStatus.OK)
                        .contentType(MediaType.APPLICATION_JSON)
                        .body(BodyInserters.fromValue(result));
        }
    };
    GatewayCallbackManager.setBlockHandler(blockRequestHandler);
}
```

### 8.5 网关规则持久化

#### 1.注释掉`POM`中的`datasource-nacos`依赖的`scope`

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
    <!--<scope>test</scope>-->
</dependency>
```

#### 2.在配置文件中添加配置

```properties
nacos.serverAddr=192.168.11.100:8848
# 如果有单独的命名空间添加此配置, 没有就不设置
# nacos.namespace=
# 如果开启了 nacos 的权限认证则需要用户名和密码，不需要再微服务中再指定
# nacos.username=nacos
# nacos.password=nacos
```

#### 3.修改`Nacos`配置类

在 src/main/java/com/alibaba/csp/sentinel/dashboard/rule 下新建`nacos`包，并创建三个子包：

```bash
rule
  --nacos
      --gateway
          --api
          --flow
      --config
```

复制 src/test/java/com/alibaba/csp/sentinel/dashboard/rule/nacos 下的`NacosConfig`、`NacosConfigUtil`类到`nacos/config`包。

然后修改`NacosConfig`：

```java
@Configuration
public class NacosConfig {
	
	@Value("${nacos.serverAddr:192.168.11.100:8848}")
	private String serverAddr;
	
	@Value("${nacos.namespace:}")
	private String namespace;
	
	@Value("${nacos.username:nacos}")
	private String username;
	
	@Value("${nacos.password:nacos}")
	private String password;
	
	
	/**
	 * 流控规则
	 * 
	 */
    @Bean
    public Converter<List<FlowRuleEntity>, String> flowRuleEntityEncoder() {
        return JSON::toJSONString;
    }

    @Bean
    public Converter<String, List<FlowRuleEntity>> flowRuleEntityDecoder() {
        return s -> JSON.parseArray(s, FlowRuleEntity.class);
    }
    
    /**
     * 授权规则
     *
     */
    @Bean
    public Converter<List<AuthorityRuleEntity>, String> authorRuleEntityEncoder() {
        return JSON::toJSONString;
    }

    @Bean
    public Converter<String, List<AuthorityRuleEntity>> authorRuleEntityDecoder() {
        return s -> JSON.parseArray(s, AuthorityRuleEntity.class);
    }
    
    /**
     * 降级规则
     *
     */
    @Bean
    public Converter<List<DegradeRuleEntity>, String> degradeRuleEntityEncoder() {
        return JSON::toJSONString;
    }

    @Bean
    public Converter<String, List<DegradeRuleEntity>> degradeRuleEntityDecoder() {
        return s -> JSON.parseArray(s, DegradeRuleEntity.class);
    }
    
    /**
     * 热点规则
     *
     */
    @Bean
    public Converter<List<ParamFlowRuleEntity>, String> paramRuleEntityEncoder() {
        return JSON::toJSONString;
    }

    @Bean
    public Converter<String, List<ParamFlowRuleEntity>> paramRuleEntityDecoder() {
        return s -> JSON.parseArray(s, ParamFlowRuleEntity.class);
    }

    /**
     * 系统规则
     *
     */
    @Bean
    public Converter<List<SystemRuleEntity>, String> systemRuleEntityEncoder() {
        return JSON::toJSONString;
    }

    @Bean
    public Converter<String, List<SystemRuleEntity>> systemRuleEntityDecoder() {
        return s -> JSON.parseArray(s, SystemRuleEntity.class);
    }
    
    /**
     * 网关API分组管理规则
     *
     */
    @Bean
    public Converter<List<ApiDefinitionEntity>, String> apiDefinitionEntityEncoder() {
        return JSON::toJSONString;
    }

    @Bean
    public Converter<String, List<ApiDefinitionEntity>> apiDefinitionEntityDecoder() {
        return s -> JSON.parseArray(s, ApiDefinitionEntity.class);
    }

    /**
     * 网关流控规则
     *
     */
    @Bean
    public Converter<List<GatewayFlowRuleEntity>, String> gatewayFlowRuleEntityEncoder() {
        return JSON::toJSONString;
    }

    @Bean
    public Converter<String, List<GatewayFlowRuleEntity>> gatewayFlowRuleEntityDecoder() {
        return s -> JSON.parseArray(s, GatewayFlowRuleEntity.class);
    }

    @Bean
    public ConfigService nacosConfigService() throws Exception {
    	Properties properties = new Properties();
        properties.put(PropertyKeyConst.SERVER_ADDR, serverAddr);
        if (StringUtils.isNotBlank(namespace)){
          properties.put(PropertyKeyConst.NAMESPACE, namespace);
        }
        properties.put(PropertyKeyConst.USERNAME, username);
        properties.put(PropertyKeyConst.PASSWORD, password);
        return ConfigFactory.createConfigService(properties);
    }
}
```

修改`NacosConfigUtil`类：

```java
/** 流控规则，已经有了 */
public static final String FLOW_DATA_ID_POSTFIX = "-flow-rules";
/** 降级规则 */
public static final String DEGRADE_DATA_ID_POSTFIX = "-degrade-rules";
/** 系统保护规则 */
public static final String SYSTEM_DATA_ID_POSTFIX = "-system-rules";
/** 访问控制规则 */
public static final String AUTHORITY_DATA_ID_POSTFIX = "-authority-rules";
/** 网关流控规则 */
public static final String GATEWAY_FLOW_DATA_ID_POSTFIX = "-gw-flow-rules";
/** 网关API分组管理规则 */
public static final String GATEWAY_API_DATA_ID_POSTFIX = "-gw-api-group-rules";
```

#### 4.创建Provider和Publisher

仿照`test`包下的`FlowRuleNacosProvider`和`FlowRuleNacosPublisher`，在`nacos/gateway/api`包下创建`GatewayApiRuleNacosProvider`和`GatewayApiRuleNacosPublisher`，仅需更改==**类名**==、==**泛型类型**==和==**后缀 Constant**==。

```java
/**
 * 拉取Nacos中存储的网关分组管理规则配置信息
 *
 */
@Component("gatewayApiRuleNacosProvider")
public class GatewayApiRuleNacosProvider implements DynamicRuleProvider<List<ApiDefinitionEntity>> {

    @Autowired
    private ConfigService configService;
    @Autowired
    private Converter<String, List<ApiDefinitionEntity>> converter;

    @Override
    public List<ApiDefinitionEntity> getRules(String appName) throws Exception {
        String rules = configService.getConfig(appName + NacosConfigUtil.GATEWAY_API_DATA_ID_POSTFIX,
            NacosConfigUtil.GROUP_ID, 3000);
        if (StringUtil.isEmpty(rules)) {
            return new ArrayList<>();
        }
        return converter.convert(rules);
    }
}
```

```java
@Component("gatewayApiRuleNacosPublisher")
public class GatewayApiRuleNacosPublisher implements DynamicRulePublisher<List<ApiDefinitionEntity>> {

    @Autowired
    private ConfigService configService;
    @Autowired
    private Converter<List<ApiDefinitionEntity>, String> converter;

    @Override
    public void publish(String app, List<ApiDefinitionEntity> rules) throws Exception {
        AssertUtil.notEmpty(app, "app name cannot be empty");
        if (rules == null) {
            return;
        }
        configService.publishConfig(app + NacosConfigUtil.GATEWAY_API_DATA_ID_POSTFIX,
            NacosConfigUtil.GROUP_ID, converter.convert(rules));
    }
}
```

同样的，在`nacos/gateway/flow`包下创建`GatewayFlowRuleNacosProvider`和`GatewayFlowRuleNacosPublisher`。

#### 5.修改`GatewayApiController`接口

仿照`controller.v2.FlowControllerV2`修改 src/main/java/com/alibaba/csp/sentinel/dashboard/controller/gateway 下的`GatewayApiController`：

```java
/*
 * 添加 ruleProvider 和 rulePublisher
 * @Autowired private SentinelApiClient sentinelApiClient;
 */

@Autowired
@Qualifier("gatewayApiRuleNacosProvider")
private DynamicRuleProvider<List<ApiDefinitionEntity>> ruleProvider;

@Autowired
@Qualifier("gatewayApiRuleNacosPublisher")
private DynamicRulePublisher<List<ApiDefinitionEntity>> rulePublisher;

...

@GetMapping("/list.json")
@AuthAction(AuthService.PrivilegeType.READ_RULE)
public Result<List<ApiDefinitionEntity>> queryApis(String app, String ip, Integer port) {
    ...
    try {
        // 获得 rules
        List<ApiDefinitionEntity> rules = ruleProvider.getRules(app);
        if (!CollectionUtils.isEmpty(rules)) {
            for (ApiDefinitionEntity entity : rules) {
                entity.setApp(app);
                entity.setIp(ip);
                entity.setPort(port);
            }
        }
        rules = repository.saveAll(rules);
        return Result.ofSuccess(rules);
    } catch (Throwable throwable) {
        logger.error("Error when querying flow rules", throwable);
        return Result.ofThrowable(-1, throwable);
    }
}


@PostMapping("/new.json")
@AuthAction(AuthService.PrivilegeType.WRITE_RULE)
public Result<ApiDefinitionEntity> addApi(HttpServletRequest request, @RequestBody AddApiReqVo reqVo) {
    try {
        entity = repository.save(entity);
        // 发布规则
        publishRules(app);
    } catch (Throwable throwable) {
        logger.error("add gateway api error:", throwable);
        return Result.ofThrowable(-1, throwable);
    }
    /*
     * if (!publishApis(app, ip, port)) {
     * logger.warn("publish gateway apis fail after add"); }
     */

    return Result.ofSuccess(entity);
}

@PostMapping("/save.json")
@AuthAction(AuthService.PrivilegeType.WRITE_RULE)
public Result<ApiDefinitionEntity> updateApi(@RequestBody UpdateApiReqVo reqVo) {
    ...
    try {
        entity = repository.save(entity);
        // 发布规则
        publishRules(app);
    } catch (Throwable throwable) {
        logger.error("update gateway api error:", throwable);
        return Result.ofThrowable(-1, throwable);
    }

    /*
     * if (!publishApis(app, entity.getIp(), entity.getPort())) {
     * logger.warn("publish gateway apis fail after update"); }
     */

    return Result.ofSuccess(entity);
}

@PostMapping("/delete.json")
@AuthAction(AuthService.PrivilegeType.DELETE_RULE)
public Result<Long> deleteApi(Long id) {
    ...
    try {
        repository.delete(id);
        // 发布规则
        publishRules(oldEntity.getApp());
    } catch (Throwable throwable) {
        logger.error("delete gateway api error:", throwable);
        return Result.ofThrowable(-1, throwable);
    }

    /*
     * if (!publishApis(oldEntity.getApp(), oldEntity.getIp(), oldEntity.getPort()))
     * { logger.warn("publish gateway apis fail after delete"); }
     */

    return Result.ofSuccess(id);
}
```

- 类似的，仿照`GatewayApiController`修改`GatewayFlowRuleController`

#### 6.修改控制台页面

修改`src/main/webapp/resources/gulpfile.js`：

```html
open('http://localhost:8080/index_dev.htm') -> open('http://localhost:8080/index.htm')
```

#### 7.修改客户端配置

- 添加`sentinel-datasource-nacos`依赖

- 在 nacos 中配置 gateway-config 的配置文件

  ```yaml
  server:
    port: 8011
  spring:
    cloud:
      gateway:
        routes:
        - id: order_route
          uri: lb://order-server
          predicates:
          - Path=/order/add,/simpleError,/getError
      sentinel:
        eager: true
        transport:
          dashboard: localhost:8080
          port: 8720
        scg:
          fallback:
            mode: response
            response-status: 426
            response-body: '{"code": 426,"message": "！"}'
        # 配置 datasource，用于客户端连接 nacos 确认规则
        datasource:
          flow-rule:
            nacos:
              serverAddr: 192.168.11.100:8848
              dataId: ${spring.application.name}-gw-flow-rules
              ruleType: flow
              username: nacos
              password: nacos
  ```

### 8.6 其他规则持久化

#### 1.创建其他规则目录类

```bash
rule
  --nacos
      --authority
          --AuthorityRuleNacosProvider
          --AuthorityRuleNacosPublisher
      --degrade
          --DegradeRuleNacosProvider
          --DegradeRuleNacosPublisher
      --flow
          --FlowRuleNacosProvider
          --FlowRuleNacosPublisher
      --param
          --ParameRuleNacosProvider
          --ParameRuleNacosPublisher
      --system
          --SystemRuleNacosProvider
          --SystemRuleNacosPublisher
      --gateway
          --api
              --GatewayApiRuleNacosProvider
              --GatewayApiRuleNacosPublisher
          --flow
              --GatewayFlowRuleNacosProvider
              --GatewayFlowRuleNacosPublisher
      --config
          --NacosConfig
          --NacosConfigUtil
```

#### 2.创建Provider和Publisher

仿照`test`包下的`FlowRuleNacosProvider`和`FlowRuleNacosPublisher`，为所有规则创建相应的 Provider 和 Publisher。

#### 3.修改Controller

修改`AuthorityRuleController`、`DegradeController`、`FlowControllerV1`、`ParamFlowRuleController`、`SystemController`。

#### 4.修改客户端配置

在客户端的配置中添加相应的规则`datasource`。

### 8.7 扩展整合Sentinel熔断

即使在服务中添加`@Sentinel`也无法被 gateway 感知到，此时可以在 gateway 中添加一个`WebFilter`来实现错误响应码熔断：

```java
@Component
class MyGatewayFilter implements WebFilter {

    Logger logger = LoggerFactory.getLogger(this.getClass());

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        ServerHttpResponse response = exchange.getResponse();
        ServerHttpResponseDecorator newResponse = new ServerHttpResponseDecorator(response) {
            @Override
            public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {
                // get match route id
                Route route = (Route) exchange.getAttributes().get(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
                String id = route.getId();
                Integer statusCode = response.getRawStatusCode();
                int lowCode = HttpStatus.BAD_REQUEST.value();
                int highCode = HttpStatus.NETWORK_AUTHENTICATION_REQUIRED.value();
                if (null != statusCode
                        && 0 != lowCode
                        && statusCode >= lowCode
                        && statusCode <= highCode) {
                    Entry entry = null;
                    try {
                        entry = SphU.entry(id, EntryType.OUT, 0);
                        // 统计发生错误的次数
                        Tracer.trace(new Exception("error"));
                    } catch (BlockException e) {
                        logger.error("an error occurs: {}", e.getCause().getMessage());
                    } finally {
                        if (entry != null)
                            entry.close();
                    }
                }
                return super.writeWith(body);
            }
        };
        return chain.filter(exchange.mutate().response(newResponse).build());
    }
}
```

> **注意**
>
> API 网管页面中没有继承"热点"和"授权"，这两个规则需要针对单个服务进行[设置](第03章_Sentinel.md#5.3-热点参数流控)。

## 10.网关高可用

可以同时启动多个 Gateway 实例，使用 LVS 或者 Nginx 进行负载。

## 11.动态加载网关路由

1. **Nacos 连接配置类**

   ```java
   @Component
   public class GatewayNacosProperties {
   
       public static String namespace;
   
       public static String serverAddr;
   
       public static String nacosRouteDataId;
   
       public static String nacosRouteGroup;
   
       public static String username;
   
       public static String password;
   
       public static final long DEFAULT_TIMEOUT = 30000;
   
   
   
       @Value("${spring.cloud.nacos.namespace:}")
       public  void setNamespace(String namespace) {
           GatewayNacosProperties.namespace = namespace;
       }
   
   
       @Value("${spring.cloud.nacos.server-addr}")
       public void setServerAddr(String serverAddr) {
           GatewayNacosProperties.serverAddr = serverAddr;
       }
   
   
       @Value("${spring.cloud.nacos.customConfig.data-id}")
       public  void setNacosRouteDataId(String nacosRouteDataId) {
           GatewayNacosProperties.nacosRouteDataId = nacosRouteDataId;
       }
   
   
       @Value("${spring.cloud.nacos.customConfig.group:DEFAULT_GROUP}")
       public  void setNacosRouteGroup(String nacosRouteGroup) {
           GatewayNacosProperties.nacosRouteGroup = nacosRouteGroup;
       }
   
       @Value("${spring.cloud.nacos.username}")
       public  void setUsername(String username) {
           GatewayNacosProperties.username = username;
       }
   
   
       @Value("${spring.cloud.nacos.password}")
       public void setPassword(String password) {
           GatewayNacosProperties.password = password;
       }
   }
   ```

2. **配置文件中配置信息**

   ```yaml
   spring:
     application:
       name: gateway-config
     cloud:
       nacos:
         server-addr: 192.168.11.100:8848
         username: nacos
         password: nacos
         config:
           file-extension: yaml
         # 自定义配置信息
         customConfig:
           data-id: gate-way-route
           # group:
     config:
       import: nacos:${spring.application.name}
     main:
       web-application-type: reactive
   ```

3. **Nacos 中配置路由信息**

   ```json
   [
       {
           "id": "gateway",
           "uri": "lb://order-server",
           "order": 0,
           "predicates": [
               {
                   "args": {
                       "pattern1": "/normal",
                       "pattern2": "/simpleError"
                   },
                   "name": "Path"
               }
           ]
       }
   ]
   ```

4. **Nacos 连接监听类**

   ```java
   @Slf4j
   @Component
   @DependsOn({"gatewayNacosProperties"})
   public class GatewayRouteNaocsConnector {
   
       // nacos 配置服务
       private ConfigService configService;
   
       @Autowired
       DynamicRouterPublisher dynamicRouterPublisher;
   
       @PostConstruct
       public void init() {
           log.info("gateway route init...");
   
           try {
               configService=initConfigService();
               if (null  == configService){
                   log.error("init config service fail");
                   return;
               }
               //通过 Nacos Config 并指定路由配置路径去获取路由配置
               String config = configService.getConfig(
                       GatewayNacosProperties.nacosRouteDataId,
                       GatewayNacosProperties.nacosRouteGroup,
                       GatewayNacosProperties.DEFAULT_TIMEOUT
               );
   
               log.info("get current gateway config from NACOS :[{}]",config);
               List<RouteDefinition> definitionList = JSON.parseArray(config, RouteDefinition.class);
   
               if (CollectionUtils.isNotEmpty(definitionList)){
                   for (RouteDefinition routeDefinition : definitionList) {
                       log.info("init gateWay config :[{}]",routeDefinition.toString());
                       dynamicRouterPublisher.addRouteDefinition(routeDefinition);
                   }
               }
           } catch (Exception e) {
               log.error("gateway route has some error:[{}]",e.getMessage(),e);
           }
   
           //设置监听器
           dynamicRouteByNacosListener(GatewayNacosProperties.nacosRouteDataId,
                   GatewayNacosProperties.nacosRouteGroup);
       }
   
       /**
        * 连接 nacos
        */
       private ConfigService initConfigService() {
           Properties properties = new Properties();
           properties.setProperty("serverAddr", GatewayNacosProperties.serverAddr);
           if (StringUtils.isNotBlank(GatewayNacosProperties.namespace)) {
               properties.setProperty("namespace", GatewayNacosProperties.namespace);
           }
           properties.setProperty("username", GatewayNacosProperties.username);
           properties.setProperty("password", GatewayNacosProperties.password);
   
           try {
               return configService = NacosFactory.createConfigService(properties);
           } catch (NacosException e) {
               log.error("init gateway nacos config error:[{}]", e.getMessage(), e);
               return null;
           }
       }
   
       /**
        * 监听 Nacos下发的动态路由配置
        *
        */
       private void dynamicRouteByNacosListener(String dataId, String group) {
           try {
               configService.addListener(dataId, group, new Listener() {
                   //可以自定义线程池来操作
                   @Override
                   public Executor getExecutor() {
                       return null;
                   }
   
                   //此时传过来的 s 就是 Nacos 中最新的配置
                   @Override
                   public void receiveConfigInfo(String s) {
                       log.info("start to update config:[{}]", s);
                       List<RouteDefinition> definitionList = JSON.parseArray(s, RouteDefinition.class);
                       log.info("update route :[{}]", definitionList.toString());
                       dynamicRouterPublisher.updateList(definitionList);
                   }
               });
           } catch (NacosException e) {
               log.error("dynamic update gateway config error:[{}]", e.getMessage(), e);
           }
       }
   }
   ```

5. **更新路由类**

   ```java
   @RequiredArgsConstructor
   @Service
   @Slf4j
   public class DynamicRouterPublisher implements ApplicationEventPublisherAware {
   
       private final RouteDefinitionWriter routeDefinitionWriter;
       private final RouteDefinitionLocator routeDefinitionLocator;
   
       //事件发布
       private ApplicationEventPublisher publisher;
   
   
       @Override
       public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
           //完成事件推送句柄的初始化
           this.publisher = applicationEventPublisher;
       }
   
       /**
        * 新增路由
        */
       public String addRouteDefinition(RouteDefinition definition) {
           log.info("gateway add route : [{}]", definition);
           //保存路由配置并发布
           routeDefinitionWriter.save(Mono.just(definition)).subscribe();
           //发布事件通知给Gateway,同步新增的路由定义
           this.publisher.publishEvent(new RefreshRoutesEvent(this));
   
           return "add success";
       }
   
       /**
        * 删除路由
        */
       public String deleteRouteById(String id) {
           try {
               log.info("gateway delete route id:[{}]", id);
               routeDefinitionWriter.delete(Mono.just(id)).subscribe();
               //发布事件通知给GateWay,同步更新路由定义
               this.publisher.publishEvent(new RefreshRoutesEvent(this));
               return "delete success";
   
           } catch (Exception ex) {
               log.error("gateway delete route fail:[{}]", ex.getMessage(), ex);
               return "delete fail";
           }
       }
   
       /**
        * 更新路由：删除 + 新增 = 更新
        */
       public String updateByRouteDefinition(RouteDefinition definition) {
   
           try {
               log.info("gateway update route:[{}]", definition);
               //只是删除不要去发布
               this.routeDefinitionWriter.delete(Mono.just(definition.getId()));
           } catch (Exception e) {
               return "update fail,not find route routeId" + definition.getId();
           }
           try {
               //现在可以发布了
               this.routeDefinitionWriter.save(Mono.just(definition)).subscribe();
               this.publisher.publishEvent(new RefreshRoutesEvent(this));
               return "success";
           } catch (Exception e) {
               return "update route fail";
           }
       }
   
       /**
        * 批量更新
        */
       public String updateList(List<RouteDefinition> definitions) {
   
           log.info("gateway update route:[{}]", definitions);
           //先拿到当前 gateway 中存储的路由定义
           List<RouteDefinition> routeDefinitions = routeDefinitionLocator.getRouteDefinitions().buffer().blockFirst();
           if (!CollectionUtils.isEmpty(routeDefinitions)) {
               //如果 gateway 中存在旧的路由定义，那么要清除掉
               routeDefinitions.forEach(rd -> {
                   log.info("delete route definition:[{}]", rd);
                   deleteRouteById(rd.getId());
               });
           }
           //把更新的路由定义同步到 gateway 中
           definitions.forEach(this::updateByRouteDefinition);
           return "success";
       }
   
   }
   ```

## 12.附录

### 12.1 网关相关环境

最终网关的配置文件如下：

**`application.yaml`**

```yaml
spring:
  application:
    name: gateway-config
  cloud:
    nacos:
      server-addr: 192.168.11.100:8848
      username: nacos
      password: nacos
      config:
        file-extension: yaml
      # 动态加载网关配置
      customConfig:
        data-id: gate-way-route
  config:
    import: nacos:${spring.application.name}
  main:
    web-application-type: reactive
```

**配置中心**

gateway-config

```yaml
server:
  port: 8011
spring:
  cloud:
    sentinel:
      eager: true
      transport:
        dashboard: localhost:8080
        port: 8720
      scg:  
        fallback:
          mode: response
          response-status: 427
          response-body: '{"code": 426,"message": "限流了，稍后重试！"}'
      datasource: 
        flow-rule:
          nacos:
            groupId: SENTINEL_GROUP
            serverAddr: 192.168.11.100:8848
            dataId: ${spring.application.name}-gw-flow-rules
            ruleType: flow
            username: nacos
            password: nacos
        degrade-rule:
           nacos:
            groupId: SENTINEL_GROUP
            serverAddr: 192.168.11.100:8848
            dataId: ${spring.application.name}-degrade-rules
            ruleType: degrade
            username: nacos
            password: nacos
```

gate-way-route

```json
[
    {
        "id": "gateway",
        "uri": "lb://order-server",
        "order": 0,
        "predicates": [
            {
                "args": {
                    "pattern1": "/normal",
                    "pattern2": "/simpleError"
                },
                "name": "Path"
            }
        ]
    }
]
```

网关需要引入的依赖有：

```xml
<dependencies>
    <!-- spring cloud gateway 核心 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    
    <!-- 引入sentinel进行服务降级熔断 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>
    <!-- gateway网关整合sentinel进行限流降级 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
    </dependency>

    <!-- Nacos 依赖 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>
    <!-- 规则持久化依赖 -->
    <dependency>
        <groupId>com.alibaba.csp</groupId>
        <artifactId>sentinel-datasource-nacos</artifactId>
    </dependency>
    
</dependencies>
```

### 12.2 服务相关环境

服务的配置文件如下：

`application.yaml`

```yaml
spring:
  config:
    import: nacos:order-${env},nacos:common-prop
  application:
    name: order-server
  cloud:
    nacos:
      config:
        group: my_GROUP
        file-extension: yaml
      username: nacos
      password: nacos
      server-addr: 192.168.11.100:8848
```

配置中心

```yaml
server:
  port: 8081
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    username: root
    password: root
    url: jdbc:mysql://192.168.11.100:3306/dev?rewriteBatchedStatements=true
mybatis:
  mapper-locations: classpath:mapper/*.xml
  configuration:
    map-underscore-to-camel-case: true
```

有网关的情况下服务不需要引入 Sentinel 的依赖。仅需引入以下重要组件：

```xml
<!-- Mybatis 组件 -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
</dependency>
<!-- openFeign 相关组件 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<!-- Nacos 注册中心和配置中心组件 -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
<!-- 其他组件，例如 senta 的组件 -->
<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring‐cloud‐starter‐alibaba‐seata</artifactId>
</dependency>
```

