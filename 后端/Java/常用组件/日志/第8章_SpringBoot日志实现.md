# 第8章_SpringBoot日志实现

SpringBoot 默认使用 slf4j 作为日志门面，logback 作为日志实现来记录日志。同时配置了 log4j2、jul 的桥接实现。

##  1.默认测试

SpringBoot 默认是基于 logback 的实现输出 info 级别。

```java
void test01() {
    Logger logger = LoggerFactory.getLogger(StarterTestApplicationTests.class);
    logger.trace("trace");
    logger.debug("debug");
    logger.info("info");
    logger.warn("warn");
    logger.error("error");
}
```

## 2.桥接log4j2测试

SpringBoot 默认继承了 log4j2 的桥接组件，使用 log4j2 的实现发现输出格式仍然是 logback 的实现。

```java
@Test
void test02() {
    org.apache.logging.log4j.Logger logger = LogManager.getLogger(StarterTestApplicationTests.class);
    logger.info("info");
}
```

## 3.修改基础配置

在`application.properties`文件中可以修改一些基础的配置。

```properties
# 配置记录器，指定 com.youyi 包下的日志输出级别
logging.level.com.youyi=trace
logging.pattern.console=%d{yyyy-MM-dd} [%level] -%m%n
# 配置存储 log 的文件夹，默认文件名为 spring.log
logging.file.path=D:/test/springbootlog
```

控制台输出

```bash
2023-01-01 [INFO] -info
```

## 4.修改高级配置

一些高级配置只能通过配置文件修改，直接在 resources 文件夹下放入相应的配置文件即可。

## 5.导入 log4j2

使用 log4j2 时，需要将 logback 的依赖去除，引入`spring-boot-starter-log4j2`依赖，该依赖继承了`log4j-slf4j-impl`和`log4j-core`。

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.6.11</version>
</parent>

<!-- 添加 log4j2 依赖 -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-log4j2</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <exclusions>
            <!-- 排除掉原始依赖 -->
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-logging</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
</dependencies>
```

同时加入 log4j2.xml 配置文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>

<Configuration  xlms="https://logging.apache.org/log4j/2.x/">

    <Properties>
        <Property name="pattern" value="[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5level] [%t] [%c#%M-%L] %m%n"/>
    </Properties>

    <Appenders>
        <Console name="console" target="SYSTEM_OUT">
            <PatternLayout pattern="${pattern}"/>
        </Console>
    </Appenders>

    <Loggers>
        <Logger name="com.youyi" level="info" additivity="false">
            <AppenderRef ref="console"/>
        </Logger>
        <Root level="error">
            <AppenderRef ref="console"/>
        </Root>
    </Loggers>

</Configuration>
```

测试

```java
@Test
void test() {
    Logger logger = LoggerFactory.getLogger(StarterTestApplicationTests.class);
    logger.trace("trace");
    logger.debug("debug");
    logger.info("info");
    logger.warn("warn");
    logger.error("error");
}
```

## 6.MDC链路追踪

底层实现是 `ThreadContext`。

在代码中使用 `MDC` 放入 trace 信息：

```java
Logger logger = LoggerFactory.getLogger(Demo4ApplicationTests.class);
@Test
void contextLoads() {
    MDC.put("traceId", UUID.randomUUID().toString());
    MDC.put("ts", String.valueOf(new Date().getTime()));
    logger.info("test");
}
```

在 `log4j2.xml` 配置文件的 `pattern` 中使用 `X{}` 获取 `MDC` 中的数据：

```xml
<Properties>
    <Property name="pattern" value="[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5level, %X{traceId}, %X{ts}] [%t] [%c#%M-%L] %m%n"/>
</Properties>
```

输出：

```bash
[2023-12-20 20:48:39.386] [INFO , 489aff74-d4b5-45e6-a6fe-7bb3faef4e28, 1703072919385] [main] [com.example.demo.Demo4ApplicationTests#contextLoads-21] test
```

> **注意**
>
> 使用线程池时要注意删除 MDC 中的数据。

## 7.开启父线程数据共享

**（1）log4j**

log4j 的 `ThreadContext` 默认使用 `ThreadLocal` 实现，其无法共享父线程的数据；需要开启共享时需要在 `resources` 下新建 `log4j2.component.properties`，添加以下配置：

```properties
isThreadContextMapInheritable=true
```

此时会使用 `InheritableThreadLocal` 作为 `ThreadContext`。

该文件可连通 `log4j2.xml` 一起放在依赖的包里。

> **扩展：底层原理**
>
> 其实现逻辑在 `DefaultThreadContextMap` 中：
>
> ```java 
> static ThreadLocal<Map<String, String>> createThreadLocalMap(final boolean isMapEnabled) {
>  return (ThreadLocal)(inheritableMap ? new InheritableThreadLocal<Map<String, String>>() {
>      protected Map<String, String> childValue(final Map<String, String> parentValue) {
>          return parentValue != null && isMapEnabled ? Collections.unmodifiableMap(new HashMap(parentValue)) : null;
>      }
>  } : new ThreadLocal());
> }
> 
> static void init() {
>  inheritableMap = PropertiesUtil.getProperties().getBooleanProperty("isThreadContextMapInheritable");
> }
> 
> static {
>  init();
> }
> ```
>

此时开启新的线程就能获取父线程的数据了：

```java
Logger logger = LoggerFactory.getLogger(Demo4ApplicationTests.class);
@Test
void contextLoads() {
    ThreadContext.put("traceId", UUID.randomUUID().toString());
    new Thread(() -> {
        logger.info(ThreadContext.get("traceId"));
    }).start();
}
```

**（2）通用**

可以重写 `ExecutorService` 的钩子函数，在线程执行前将 MDC 的上下文信息放入线程中：

```java
@EnableAsync
@Configuration
public class CustomExecutorConfig implements AsyncConfigurer {

    // 使用 ThreadLocal 存储 MDC 的 contextMap
    private static ThreadLocal<Map<String, String>> contextMap;

    @Bean
    ExecutorService executorService1() {
        return new ThreadPoolExecutor(2, 2, 60L, TimeUnit.SECONDS, new LinkedBlockingQueue<>(16),
            new ThreadPoolExecutor.AbortPolicy()) {
            @Override
            protected void beforeExecute(Thread t, Runnable r) {
                // 在子线程任务执行前从 ThreadLocal 中取出 MDC 上下文信息并设置到自身的上下文中
                MDC.setContextMap(contextMap.get());
                super.beforeExecute(t, r);
            }

            @Override
            protected void afterExecute(Runnable r, Throwable t) {
                // 执行完成后清理
                MDC.clear();
                contextMap.remove();
                super.afterExecute(r, t);
            }

        };
    }

    @Override
    public Executor getAsyncExecutor() {
        // 使用可继承的 ThreadLocal，否则子线程的 ThreadLocalMap 中不存在该 ThreadLocal
        contextMap = new InheritableThreadLocal<>();
        contextMap.set(MDC.getCopyOfContextMap());
        return executorService1();
    }

}
```

在主线程中设置 MDC，子线程中就可以打印出 MDC 了：

**主线程**

```java
@SpringBootTest
public class ExecutorServiceDemo {

    @Autowired
    AsyncService asyncService;

    @Test
    void demo() throws InterruptedException {
        MDC.put("trace", UUID.randomUUID().toString());
        asyncService.asyncMethod();
    }

}
```

**子线程**

```java
@Service
@Slf4j
public class AsyncService {

    @Async
    public void asyncMethod() {
	log.info("test");
    }

}
```

**日志**

```xml
 <property name="pattern" value="[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%c#%M-%L] [%X{trace}] %m%n"/>
```
