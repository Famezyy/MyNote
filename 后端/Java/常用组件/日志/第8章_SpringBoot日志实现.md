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
<!-- 添加 log4j2 依赖 -->
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
            <AppenderRef ref="console"></AppenderRef>
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

