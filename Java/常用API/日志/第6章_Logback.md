# 第6章_Logback

01、简单介绍
Logback 官网：https://logback.qos.ch/

Logback 是由 Log4j 创始人设计的又一个开源日志组件。作为流行的 Log4j 项目的继承者，在 log4j 1.x 停止的地方接手。其架构非常通用，可以在不同的情况下应用。目前分为三个模块，logback-core、logback-classic 和 logback-access 。

logback-core 是其它两个模块的基础模块。
logback-classic 是 Log4j 的一个改良版本，并原生实现了 SLF4J API，可以轻松地更换成其他日志框架，例如 log4j 1.x 或 JUL 。
logback-access 与 Tomcat 和 Jetty 等 Servlet 容器集成，以提供 HTTP 访问日志功能。可以轻松在 logback-core 上构建自己的模块。

02、组件介绍
Logback 建立在三个主要类之上 Logger： Appender 和 Layout 。
这三种类型的组件协同工作，使开发人员能够根据消息类型和级别记录消息，并在运行时控制这些消息的格式和报告位置。

2.1、记录器
Loggers（记录器）控制日志的输出级别，规则是：只输出级别不低于设定级别的日志信息。以及引用 Appenders（输出器）。
Loggers（记录器）有一个根记录器位于记录器层次结构的顶部，它从一开始就是每个层次结构的一部分。
Loggers（记录器）之间有继承关系，例如名为 “com.foo” 的记录器是名为 “com.foo.Bar” 的记录器的父级。 同样 “java” 是 “java.util” 的父级和 “java.util.Vector” 的祖先。

2.2、输出器
Appender（输出器）将日志记录请求打印到多个目的地，如控制台、文件、数据库等等。

2.3、格式器
Layout（格式器）将事件转换成字符串，将日志信息格式化并输出。在 Logback 中 Layout 对象被封装在 encoder 中。

Pattern Layout（自定义格式器）常用转换符：

转换词	描述	性能
%c	在记录事件的起源处输出记录器的名称	
%C	输出发出日志请求的调用者的完全限定类名	差
%d	用于输出记录事件的日期，如 %d{yyyy-MM-dd HH:mm:ss.SSS}	
%F / file	输出发出记录请求的 Java 源文件的文件名	差
%L / line	输出发出记录请求的行号	差
%m / msg / message	输出与日志事件关联的应用程序提供的消息	
%M / method	输出发出记录请求的方法名称	差
%n	输出平台相关的行分隔符或字符	
%p / le / level	输出日志事件的级别	
%r / relative	输出从应用程序启动到创建日志记录事件所经过的毫秒数	
%t / thread	输出生成日志事件的线程的名称	
%20c	如果记录器名称的长度少于 20 个字符，则用空格填充	
%-20c	如果记录器名称的长度少于 20 个字符，则用空格填充右侧	
%.30c	如果记录器名称超过 30 个字符，则从头部截断	
%.-30c	如果记录器名称超过 30 个字符， 则从尾部截断	


03、入门案例
导入依赖： Logback 搭配 SLF4J 日志门面使用。

```xml
<dependencies>
    <!-- log4j日志门面 -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.7.25</version>
    </dependency>
    <!-- logback-core是logback-classic的基础模块，根据maven依赖传递性，无需重复导入 -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.2.3</version>
    </dependency>
    <!-- 单元测试 -->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>
</dependencies>

```

**代码示例：**

```java
package com.logback;

import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LogbackTest {
    @Test
    public void test01() {
        Logger logger = LoggerFactory.getLogger(LogbackTest.class);
        logger.trace("trace追踪信息");
        logger.debug("debug详细信息"); // 默认级别
        logger.info("info关键信息");
        logger.warn("warn警告信息");
        logger.error("error错误信息");
    }
}
```

**运行结果：**

```bash
03:10:56.699 [main] DEBUG com.logback.LogbackTest - debug详细信息
03:10:56.701 [main] INFO com.logback.LogbackTest - info关键信息
03:10:56.701 [main] WARN com.logback.LogbackTest - warn警告信息
03:10:56.701 [main] ERROR com.logback.LogbackTest - error错误信息
```

## 04、配置文件

Logback 提供了 3 种配置文件： logback.groovy ，logback-test.xml ，logback.xml ，若都不存在则采用默认配置。

创建配置：logback.xml 文件。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- 配置自定义日志输出格式 -->
    <property name="pattern" value="[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%c#%M-%L] %m%n"/>

    <!-- 配置控制台输出器 -->
    <appender name="consoleAppender" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 配置日志字体颜色，默认黑色（System.out），红色（System.err） -->
        <target>System.err</target>
        <!-- 配置日志输出格式 -->
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!-- 引用自定义日志输出格式 -->
            <pattern>${pattern}</pattern>
        </encoder>
    </appender>

    <!-- 配置记录器 -->
    <root level="ALL">
        <!-- 引用控制台输出器 -->
        <appender-ref ref="consoleAppender"/>
    </root>

</configuration>
```

代码示例

```java
@Test
public void test01() {
    Logger logger = LoggerFactory.getLogger(LogbackTest.class);
    logger.trace("trace追踪信息");
    logger.debug("debug详细信息");
    logger.info("info关键信息");
    logger.warn("warn警告信息");
    logger.error("error错误信息");
}
```

运行结果

```bash
[2022-07-02 04:19:59.476] [TRACE] [main] [com.logback.LogbackTest#test01-12] trace追踪信息
[2022-07-02 04:19:59.478] [DEBUG] [main] [com.logback.LogbackTest#test01-13] debug详细信息
[2022-07-02 04:19:59.478] [INFO ] [main] [com.logback.LogbackTest#test01-14] info关键信息
[2022-07-02 04:19:59.479] [WARN ] [main] [com.logback.LogbackTest#test01-15] warn警告信息
[2022-07-02 04:19:59.479] [ERROR] [main] [com.logback.LogbackTest#test01-16] error错误信息
```

## 05、保存日志

保存到 log 文件，相关配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- 配置自定义日志输出格式 -->
    <property name="pattern" value="[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%c#%M-%L] %m%n"/>
    <!-- 配置日志输出目录 -->
    <property name="logDir" value="../logDir"/>

    <!-- 配置文件输出器 -->
    <appender name="fileAppender" class="ch.qos.logback.core.FileAppender">
        <!-- 配置日志输出文件 -->
        <file>${logDir}/logback.log</file>
        <!-- 配置日志输出格式 -->
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!-- 引用自定义日志输出格式 -->
            <pattern>${pattern}</pattern>
        </encoder>
    </appender>

    <!-- 配置记录器 -->
    <root level="ALL">
        <!-- 引用文件输出器-->
        <appender-ref ref="fileAppender"/>
    </root>
    
</configuration>
```

保存到 HTML 文件，相关配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- 配置自定义日志输出格式 -->
    <property name="pattern1" value="%d{yyyy-MM-dd HH:mm:ss.SSS}%-5p%t%c%M%L%m%n"/>
    <!-- 配置日志输出目录 -->
    <property name="logDir" value="../logDir"/>

    <!-- 配置HTML输出器-->
    <appender name="htmlAppender" class="ch.qos.logback.core.FileAppender">
        <!-- 配置日志输出文件 -->
        <file>${logDir}//logback.html</file>
        <!-- 配置日志输出格式 -->
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="ch.qos.logback.classic.html.HTMLLayout">
                <!-- 引用自定义日志输出格式 -->
                <pattern>${pattern1}</pattern>
            </layout>
        </encoder>
    </appender>

    <!-- 配置记录器 -->
    <root level="ALL">
        <!-- 引用HTML输出器 -->
        <appender-ref ref="htmlAppender"/>
    </root>

</configuration>
```

保存到拆分文件，相关配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- 配置自定义日志输出格式 -->
    <property name="pattern" value="[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%c#%M-%L] %m%n"/>
    <!-- 配置日志输出目录 -->
    <property name="logDir" value="../logDir"/>

    <!-- 配置拆分归档输出器 -->
    <appender name="rollingFileAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 配置日志输出文件 -->
        <file>${logDir}/rollingFile.log</file>
        <!-- 配置日志输出格式 -->
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!-- 引用自定义日志输出格式 -->
            <pattern>${pattern}</pattern>
        </encoder>
        <!-- 配置拆分规则 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!-- 按照时间和压缩格式声名文件名 -->
            <fileNamePattern>../logDir/rollingFile.%d{yyyy-MM-dd}.log%i.gz</fileNamePattern>
            <!-- 按照指定文件大小进行拆分 -->
            <maxFileSize>1KB</maxFileSize>
        </rollingPolicy>
    </appender>
    
    <!-- 配置记录器 -->
    <root level="ALL">
        <!-- 引用拆分归档输出器 -->
        <appender-ref ref="rollingFileAppender"/>
    </root>

</configuration>
```

## 06、日志过滤

过滤器写在 Appender 标签内，可以配置一个或多个，按照先后顺序执行。过滤器会对每个级别的日志设置枚举值，表示对日志的处理方式。

过滤器枚举值	描述
DENY	当前过滤器直接拒绝日志输出，不再经过后续的过滤器
NEUTRAL	当前过滤器不做任何日志处理，若后续的过滤器返回的全部都是 NEUTRAL，则进行日志输出
ACCEPT	当前过滤器直接进行日志输出，不再经过后续的过滤器。

级别过滤器	描述
level	设置日志级别
onMatch	对符合过滤级别的日志操作
onMismatch	对不符合过滤级别的日志操作

相关配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- 配置自定义日志输出格式 -->
    <property name="pattern" value="[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%c#%M-%L] %m%n"/>

    <!-- 配置使用过滤器的控制台输出器 -->
    <appender name="consoleFilterAppender" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 配置日志字体颜色，默认黑色（System.out），红色（System.err） -->
        <target>System.err</target>
        <!-- 配置日志输出格式 -->
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!-- 引用自定义日志输出格式 -->
            <pattern>${pattern}</pattern>
        </encoder>
        <!-- 配置级别过滤器 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <!-- 过滤级别 -->
            <level>ERROR</level>
            <!-- 打印符合过滤级别的日志 -->
            <onMatch>ACCEPT</onMatch>
            <!-- 屏蔽不符合过滤级别的日志 -->
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <!-- 配置记录器 -->
    <root level="ALL">
        <!-- 引用使用过滤器的控制台输出器 -->
        <appender-ref ref="consoleFilterAppender"/>
    </root>

</configuration>
```

代码示例：

```java
@Test
public void test01() {
    Logger logger = LoggerFactory.getLogger(LogbackTest.class);
    logger.trace("trace追踪信息");
    logger.debug("debug详细信息"); 
    logger.info("info关键信息");
    logger.warn("warn警告信息");
    logger.error("error错误信息");
}
```

运行结果：

```bash
[2022-07-02 09:45:54.411] [ERROR] [main] [com.logback.LogbackTest#test-16] error错误信息
```

07、异步日志
代码按照从上向下的顺序执行，只有当上面的代码全部执行完毕，你才会执行下面的代码。

由此得出会发生的问题：

只要是在记录日志，那么系统本身的功能就处于一种停滞的状态。
当日志记录完毕后，才会执行系统本身业务代码。
如果日志的记录量很大，那对于系统本身业务代码的执行效率会降低。
所以 Logback 为此提供了异步日志功能。原理就是系统为日志操作单独分配一条线程，原本执行当前方法的主线程会继续向下执行，两条线程争夺 CPU 使用权。

相关配置：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- 配置自定义日志输出格式 -->
    <property name="pattern" value="[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%c#%M-%L] %m%n"/>

    <!-- 配置控制台输出器 -->
    <appender name="consoleAppender" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 配置日志字体颜色，默认黑色（System.out），红色（System.err） -->
        <target>System.err</target>
        <!-- 配置日志输出格式 -->
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!-- 引用自定义日志输出格式 -->
            <pattern>${pattern}</pattern>
        </encoder>
    </appender>

    <!-- 配置异步输出器 -->
    <appender name="asyncAppender" class="ch.qos.logback.classic.AsyncAppender">
        <!-- 引用控制台输出器 -->
        <appender-ref ref="consoleAppender"/>
    </appender>

    <!-- 配置日志级别、控制器 -->
    <root level="ALL">
        <!-- 引用异步输出器 -->
        <appender-ref ref="asyncAppender"/>
    </root>

</configuration>
```

代码示例：

```java
@Test
public void test02() {
    Logger logger = LoggerFactory.getLogger(LogbackTest.class);
    for (int i = 0; i < 100; i++) {
        logger.trace("trace级别信息");
        logger.debug("debug级别信息");
        logger.info("info级别信息");
        logger.warn("warn级别信息");
        logger.error("error级别信息");
    }
    for (int i = 0; i < 600; i++) {
        System.out.println("——————————————");
    }
}
```

运行结果：多运行几次，可以看出日志记录和系统打印交替输出，实现了异步操作。

```bash
略...
——————————————
[2022-07-02 11:22:39.600] [TRACE] [main] [com.logback.LogbackTest#test02-23] trace级别信息
——————————————
——————————————
——————————————
[2022-07-02 11:22:39.600] [DEBUG] [main] [com.logback.LogbackTest#test02-24] debug级别信息
[2022-07-02 11:22:39.600] [INFO ] [main] [com.logback.LogbackTest#test02-25] info级别信息
[2022-07-02 11:22:39.600] [WARN ] [main] [com.logback.LogbackTest#test02-26] warn级别信息
[2022-07-02 11:22:39.600] [ERROR] [main] [com.logback.LogbackTest#test02-27] error级别信息
——————————————
略...
```

## 08、自定义记录器

相关配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- 配置自定义日志输出格式 -->
    <property name="pattern" value="[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%c#%M-%L] %m%n"/>

    <!-- 配置控制台输出器 -->
    <appender name="consoleAppender" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 配置日志字体颜色，默认黑色（System.out），红色（System.err） -->
        <target>System.err</target>
        <!-- 配置日志输出格式 -->
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!-- 引用自定义日志输出格式 -->
            <pattern>${pattern}</pattern>
        </encoder>
    </appender>
    
	<!-- 配置自定义记录器 -->
    <logger name="com.logback" level="info" additivity="false">
        <appender-ref ref="consoleAppender"/>
    </logger>
    
</configuration>
```

代码示例：

```java
@Test
public void test01() {
    Logger logger = LoggerFactory.getLogger(LogbackTest.class);
    logger.trace("trace追踪信息");
    logger.debug("debug详细信息"); 
    logger.info("info关键信息");
    logger.warn("warn警告信息");
    logger.error("error错误信息");
}
```

运行结果：

```bash
[2022-07-02 11:31:12.659] [INFO ] [main] [com.logback.LogbackTest#test01-14] info关键信息
[2022-07-02 11:31:12.661] [WARN ] [main] [com.logback.LogbackTest#test01-15] warn警告信息
[2022-07-02 11:31:12.661] [ERROR] [main] [com.logback.LogbackTest#test01-16] error错误信息
```

