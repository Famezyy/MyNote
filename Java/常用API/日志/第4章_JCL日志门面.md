# 第4章_JCL日志门面
## 1.简介
JCL（ Jakarta Commons Logging ），是 Apache 提供的一个 通用日志 API。用户可以自由选择第三方的日志组件作为具体实现，像 Log4j 或 JDK 自带的 JUL。

common-logging 会通过动态查找的机制，在程序运行时自动找出真正使用的日志框架。其内部有一个 Simple logger 的简单实现，但是功能很弱，所以 common-logging 通常都是配合着 Log4j 以及其他日志框架来使用。

使用它的好处就是，代码依赖是 common-logging 而非 Log4j 或 JUL 的 API， 避免了和具体日志框架 API 的直接耦合。也就是面向接口开发，**不再依赖具体的实现类**，可以根据实际需求，灵活的切换日志框架。统一的 API，统一的配置管理便于项目日志的维护工作。

JCL 有两个基本的抽象类：Log（日志记录器）、LogFactory（日志工厂，负责创建 Log 实例）。

## 2.入门案例
导入依赖
```xml
<dependencies>
    <!-- jcl日志门面 -->
    <dependency>
        <groupId>commons-logging</groupId>
        <artifactId>commons-logging</artifactId>
        <version>1.2</version>
    </dependency>
</dependencies>
```
```java
package com.jcl;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.junit.Test;

public class JclTest {
    @Test
    public void test01() {
        // JCL使用原则：有Log4j则优先使用，没有任何第三方日志框架则默认使用JUL
        Log log = LogFactory.getLog(JclTest.class);
        log.info("info信息");
    }
}
```
运行结果： 可以看出使用了 JDK 的 JUL 日志框架。
```bash
七月 01, 2022 7:52:12 上午 com.jcl.JclTest test01
信息: info信息
```
添加依赖：
```xml
<!-- log4j日志框架 -->
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
</dependency>
```
创建配置： log4j.properties 文件
```properties
#配置日志级别，引用控制器
log4j.rootLogger=INFO,console
#配置控制台输出器
log4j.appender.console=org.apache.log4j.ConsoleAppender
#配置自定义格式器
log4j.appender.console.layout=org.apache.log4j.PatternLayout
#配置自定义转换模式
log4j.appender.console.layout.conversionPattern=[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%-4rms] [%c#%M-%L] %m%n
```
再次运行：可以看出使用了 Log4j 日志框架。
```bash
[2022-07-01 07:53:33.453] [INFO ] [main] [0   ms] [com.jcl.JclTest#test01-13] info信息
```
## 3.源码
查看源码可以发现，Log 接口主要有 4 个重要的实现类：
- JDK13
- JDK14：默认的`java.util.logging`
- Log4j：集成的 Log4j
- Simple：JCL 自带的实现类

```java
public abstract class LogFactory {
    public static Log getLog(Class<?> clazz) {
        return getLog(clazz.getName());
    }
    public static Log getLog(String name) {
        // 创建 Log
        return LogAdapter.createLog(name);
    }
}

final class LogAdapter {

    // 在类加载时查找是否继承了相应的 Logger
    static {
        if (isPresent("org.apache.logging.log4j.spi.ExtendedLogger")) {
            if (isPresent("org.apache.logging.slf4j.SLF4JProvider") && isPresent("org.slf4j.spi.LocationAwareLogger")) {
                logApi = LogAdapter.LogApi.SLF4J_LAL;
            } else {
                logApi = LogAdapter.LogApi.LOG4J;
            }
        } else if (isPresent("org.slf4j.spi.LocationAwareLogger")) {
            logApi = LogAdapter.LogApi.SLF4J_LAL;
        } else if (isPresent("org.slf4j.Logger")) {
            logApi = LogAdapter.LogApi.SLF4J;
        } else {
            logApi = LogAdapter.LogApi.JUL;
        }
    }
    
    private static boolean isPresent(String className) {
        try {
            Class.forName(className, false, LogAdapter.class.getClassLoader());
            return true;
        } catch (ClassNotFoundException var2) {
            return false;
        }
    }

    public static Log createLog(String name) {
        switch(logApi) {
            case LOG4J:
                return LogAdapter.Log4jAdapter.createLog(name);
            case SLF4J_LAL:
                return LogAdapter.Slf4jAdapter.createLocationAwareLog(name);
            case SLF4J:
                return LogAdapter.Slf4jAdapter.createLog(name);
            default:
                return LogAdapter.JavaUtilAdapter.createLog(name);
        }
    }
}
```