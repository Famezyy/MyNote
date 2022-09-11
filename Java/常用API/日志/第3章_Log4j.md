# 第3章_Log4j
## 3.1 简介
Log4j（Log For Java）是一个 Apache 的开源项目，我们可以控制日志信息输送的目的地是控制台、文件、GUI 组件，甚至是套接口服务器，NT 的事件记录器、UNIX Syslog 守护进程等；我们也可以控制每一条日志的输出格式，通过定义每一条日志信息的级别，我们能够更加细致地控制日志的生成过程，而这些都可以通过一个配置文件来灵活地进行配置，不需要修改代码。
## 3.2 Log4j组件介绍
Log4j 主要由 Loggers（日志记录器）、Appenders（输出控制器）和 Layout（日志格式化器）组成，其中 Loggers 控制日志的输出以及输出级别（JUL 对于输出级别单独使用 Level）；Appenders 指定日志的输出方式（输出到控制台、文件等）；Layout 控制日志信息的输出格式。
### 1.Loggers
日志记录器，负责收集处理日志记录，实例的命名就是类的全限定名，大小写敏感，命名有继承机制，例如：name 为 com.youyi.zhao 的 logger 会继承 `com.youyi`Log4j 中有一个特殊的 logger 叫做"root"。它是所有 logger 的根，所有其他的 logger 都会直接或间接地继承 root。root logger 可以用`Logger.getRootLogger()`获取。自 log4j 1.2 以来，Logger 类已经取代了 Category 类。

| 级别（由低至高） | 描述                               |
| ---------------- | ---------------------------------- |
| ALL              | 打开所有日志记录开关；是最低等级的 |
| TRACE            | 输出追踪信息；一般情况下并不会使用 |
| DEBUG（默认）    | 输出调试信息；打印些重要的运行信息 |
| INFO             | 输出提示信息；突出应用程序运行过程 |
| WARN             | 输出警告信息；会出现潜在错误的情况 |
| ERROR            | 输出错误信息；不影响系统的继续运行 |
| FATAL            | 输出致命错误；会导致应用程序的退出 |
| OFF              | 关闭所有日志记录开关；是最高等级的 |

DEBUG、INFO、WARN、ERROR……级别是分大小的：DEBUG < INFO < WARN < ERROR，分别用来指定这条日志信息的重要程度，Log4j 输出日志的规则是：只输出级别不低于设定级别的日志信息，假定级别为 INFO，则 INFO、WARN、ERROR 级别的日志信息都会输出。
### 2.Appenders
Appender（输出器）通常只负责将日志信息写入目标目的地。将格式化日志信息的责任委托给 Layout（格式器）。定义一个名字以便被 Loggers（记录器）引用。

| 组件                                           | 描述                                   |
| ---------------------------------------------- | -------------------------------------- |
| ConsoleAppender（控制台输出器）                | 将日志信息输出到控制台                 |
| FileAppender（文件输出器）                     | 将日志信息写入指定文件                 |
| DailyRollingFileAppender（按天拆分文件输出器） | 将日志信息写入指定文件，每天一个新文件 |
| RollingFileAppender（拆分文件输出器）          | 将日志信息写入多个文件                 |
| JDBCAppender（数据库输出器）                   | 将日志信息写入数据库表                 |
### 3.Layouts
Layout（格式器）接收 Appender（输出器）的日志信息将其格式化为满足任何消费日志事件需求的样式。

| 组件                          | 描述                                             |
| ----------------------------- | ------------------------------------------------ |
| HTMLLayout                    | 格式化日志输出为 HTML                            |
| SimpleLayout（简单格式器）    | 将日志信息输出为简单格式，默认为 INFO 级别的消息 |
| PatternLayout（自定义格式器） | 根据自定义的转换模式，返回格式化后的日志信息结果 |

Pattern Layout（自定义格式器）转换符：

可以在`%`与字符之间加上修饰符来控制最小宽度、最大宽度和文本的对齐方式，如：

- `%5c`：输出全类名，最小宽度是 5，默认情况下右对齐
- `%-5c`：输出全类名，最小宽度 5，左对齐，会有空格
- `%.5c`：最大宽度 5，右对齐，超过则将多余的字符截断，宽度小于 5 也不会有空格
- `%20.30c`：小于 20 补足空格，大于 30 则将左边多余的字符阶段，右对齐


| 转换符 | 描述                                                         | 性能 |
| ------ | ------------------------------------------------------------ | ---- |
| %c     | 用于输出打印语句所属的类的全类名                             |      |
| %d     | 用于输出记录事件的日期，默认为 ISO8610，也可指定样式，如 %d{yyyy-MM-dd HH:mm:ss.SSS} |      |
| %F     | 用于输出日志消息产生时所在的文件名称                         | 差   |
| %l     | 用于输出产生日志事件的调用者的位置信息，包括类名、线程、及在代码中的行数。如：Test.main(Test.java:10) | 差   |
| %L     | 用于输出发出记录请求的行号                                   | 差   |
| %m     | 用于输出日志产生的信息                                       |      |
| %M     | 用于输出发出日志请求的方法名称                               | 差   |
| %n     | 输出平台相关的行分隔符或字符                                 |      |
| %p     | 用于输出日志事件的优先级，即 DEBUG、INFO 等                  |      |
| %r     | 用于输出从启动应用到输出该 log 信息所经过的毫秒数            |      |
| %t     | 用于输出生成日志事件的线程名称                               |      |
| %%     | 用于输出一个百分号                                           |      |

## 3.3 入门案例
导入依赖：
```xml
<dependencies>
    <dependency>
        <groupId>log4j</groupId>
        <artifactId>log4j</artifactId>
        <version>1.2.17</version>
    </dependency>
</dependencies>
```
代码示例：
```java
import org.apache.log4j.BasicConfigurator;
import org.apache.log4j.Logger;
import org.junit.Test;

public class Log4jTest {
    @Test
    public void test01() {
        BasicConfigurator.configure(); //加载初始化配置
        Logger logger = Logger.getLogger(Log4jTest.class);
        logger.trace("trace信息");
        logger.debug("debug信息"); //默认级别
        logger.info("info信息");
        logger.warn("warn信息");
        logger.error("error信息");
        logger.fatal("fatal信息");
    }
}
```
运行结果：
```bash
0 [main] DEBUG com.Log4jTest  - debug信息
1 [main] INFO com.Log4jTest  - info信息
1 [main] WARN com.Log4jTest  - warn信息
1 [main] ERROR com.Log4jTest  - error信息
1 [main] FATAL com.Log4jTest  - fatal信息
```
### 3.4 配置文件
对于`xml`配置文件，底层使用的是`w3c`包来加载，对于`properties`配置文件，底层使用的是`Properties`类来加载。
#### 1.根节点的配置
观察源码`BasicConfigurator.configure()`可以发现：

- 创建根节点对象：`Logger root = Logger.getRootLogger()`
- 添加 ConsoleAppender 对象，指定了格式化输出

据此，我们发现需要在配置文件中配置 Logger、Appender、Layout 这 3 个组件。

分析`Logger.getLogger(Log4jTest.class)`可以发现是通过 LogManager 得到了 Logger：`LogManager.getLogger(clazz.getName())`。在 LogManager 中定义了很多常量，来表示不同后缀的配置文件，最常使用的是 log4j.properties。

继续观察 LogManager，可以发现在静态代码块中加载了配置文件：`Loader.getResource("log4j.properties")`。可以发现是从类路径下加载该配置文件的，对于 Maven 工程，是放在 resources 路径下的。

加载配置时，执行了`OptionConverter.selectAndConfigure(url, configuratorClassName, getLoggerRepository())`，加载属性文件时，执行了`configurator = new PropertyConfigurator()`，进入到 PropertyConfigurator 中可以发现定义了许多前缀常量，这些常量信息就是我们在 properties 属性文件中的各种属性配置项，其中如下两行信息是必须要进行配置的：
```java
static final String ROOT_LOGGER_PREFIX = "log4j.rootLogger.";
static final String APPENDER_PREFIX = "log4j.appender.";
```
```properties
log4j.rootLogger=日志级别,输出器名称[,输出器名称...]
log4j.appender.[自定义输出器名称]=输出方式
log4j.appender.[自定义输出器名称].layout=输出格式
```
创建配置：log4j.properties 文件
```properties
#配置日志级别,输出器
log4j.rootLogger=INFO,console
#配置控制台输出器
log4j.appender.console=org.apache.log4j.ConsoleAppender
#配置简单格式器
log4j.appender.console.layout=org.apache.log4j.SimpleLayout

```
代码示例：
```java
@Test
public void test02() {
    // 已有配置文件，无需再加载初始化配置
    //BasicConfigurator.configure();
    Logger logger = Logger.getLogger(Log4jTest.class);
    logger.trace("trace信息");
    logger.debug("debug信息");
    logger.info("info信息");
    logger.warn("warn信息");
    logger.error("error信息");
    logger.fatal("fatal信息");
}
```
运行结果：
```bash
INFO - info信息
WARN - warn信息
ERROR - error信息
FATAL - fatal信息
```
#### 2.输出详细信息
Logger 是记录系统的日志，而 LogLog 是用来记录 Logger 的日志。查看源码可以发现，LogManager 的`getLoggerRepository()`中调用了`LogLog.debug(msg, ex)`，LogLog 会使用 debug 级别的输出为我们展示日志输出的详细信息。

在`LogLog.debug(msg, ex)`方法中，首先通过`if(debugEnabled && !quietMode)`判断是否启用该功能，`!quietMode`默认开启，`debugEnabled`默认关闭，所以我们只需要设置`debugEnabled`为 true 即可。
```java
@Test
public void test03() {
    LogLog.setInternalDebugging(true);
    Logger logger = Logger.getLogger(Log4jTest.class);
    logger.trace("trace信息");
    logger.debug("debug信息");
    logger.info("info信息");
    logger.warn("warn信息");
    logger.error("error信息");
    logger.fatal("fatal信息");
}
```
输出
```bash
log4j: Trying to find [log4j.xml] using context classloader sun.misc.Launcher$AppClassLoader@73d16e93.
log4j: Trying to find [log4j.xml] using sun.misc.Launcher$AppClassLoader@73d16e93 class loader.
log4j: Trying to find [log4j.xml] using ClassLoader.getSystemResource().
log4j: Trying to find [log4j.properties] using context classloader sun.misc.Launcher$AppClassLoader@73d16e93.
log4j: Using URL [file:/C:/Users/APD-43164863/eclipse-workspace/com.youyi.zhao/target/classes/log4j.properties] for automatic log4j configuration.
log4j: Reading configuration from URL file:/C:/Users/APD-43164863/eclipse-workspace/com.youyi.zhao/target/classes/log4j.properties
log4j: Parsing for [root] with value=[INFO,console].
log4j: Level token is [INFO].
log4j: Category root set to INFO
log4j: Parsing appender named "console".
log4j: Parsing layout options for "console".
log4j: End of parsing for "console".
log4j: Parsed "console" options.
log4j: Finished configuring.
INFO - info信息
WARN - warn信息
ERROR - error信息
FATAL - fatal信息
```
#### 3.自定义格式
PatternLayout 是使用最多的方式，其中`setConversionPattern()`是核心。
修改配置：
```properties
# 配置日志级别,引用输出器
log4j.rootLogger=INFO,console

########## 控制台输出器 ##########
# 配置控制台输出器
log4j.appender.console=org.apache.log4j.ConsoleAppender
# 配置自定义格式器
log4j.appender.console.layout=org.apache.log4j.PatternLayout
# 配置自定义转换模式
log4j.appender.console.layout.conversionPattern=[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%-4rms] [%c] %M %L %m%n
```
代码示例：
```java
@Test
public void test04() {
    Logger logger = Logger.getLogger(Log4jTest.class);
    logger.trace("trace信息");
    logger.debug("debug信息");
    logger.info("info信息");
    logger.warn("warn信息");
    logger.error("error信息");
    logger.fatal("fatal信息");
}
```
运行结果：
```bash
[2022-06-30 21:19:45.421] [INFO ] [main] [0   ms] [com.log4j.Log4jTest] test02 26 info信息
[2022-06-30 21:19:45.406] [WARN ] [main] [985 ms] [com.log4j.Log4jTest] test02 27 warn信息
[2022-06-30 21:19:45.410] [ERROR] [main] [989 ms] [com.log4j.Log4jTest] test02 28 error信息
[2022-06-30 21:19:45.416] [FATAL] [main] [995 ms] [com.log4j.Log4jTest] test02 29 fatal信息
```
### 3.5 日志持久化
将日志输出到文件中，可以同时配置控制台和文件的输出方式。

查看 FileAppender 源码可以发现有两个比较重要的属性。虽然我们不需要去定义：`fileAppend`表示是否追加信息，默认 true，`bufferSize`表示缓冲区的大小，默认为 8192。
```properties
#配置日志级别,输出控制器
log4j.rootLogger=INFO,file

########## 文件输出器 ##########
# 配置文件输出器
# file 是自定义的输出器名称
log4j.appender.file=org.apache.log4j.FileAppender
# 配置自定义格式器
log4j.appender.file.layout=org.apache.log4j.PatternLayout
# 配置自定义转换模式
log4j.appender.file.layout.conversionPattern=[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%-4rms] [%c] %M %L %m%n
# 配置文件路径及名称
log4j.appender.file.file=../logDir/file.log
# 配置字符编码
log4j.appender.file.encoding=UTF-8
```
**按照文件大小拆分`RollingFileAppender`**

相关配置：
```properties
# 配置日志级别,输出控制器
log4j.rootLogger=INFO,rollingFile

########## 拆分文件输出器 ##########
# 配置拆分文件输出器
log4j.appender.rollingFile=org.apache.log4j.RollingFileAppender
# 配置自定义格式器
log4j.appender.rollingFile.layout=org.apache.log4j.PatternLayout
# 配置自定义转换模式
log4j.appender.rollingFile.layout.conversionPattern=[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%-4rms] [%c] %M %L %m%n
# 配置文件路径及名称
log4j.appender.rollingFile.file=../logDir/rollingFile.log
# 配置字符编码
log4j.appender.rollingFile.encoding=UTF-8
# 配置拆分文件大小
log4j.appender.rollingFile.maxFileSize=1MB
# 配置拆分文件数量
log4j.appender.rollingFile.maxBackupIndex=2

# 若文件大小超过1KB，则生成另外一个文件，数量最多2个。
# 若2个文件不够用，则按照时间来进行覆盖，保留新的覆盖旧的。
```
**按照时间进行拆分--`DailyRollingFileAppender`**

相关配置：
```properties
# 配置日志级别,输出控制器
log4j.rootLogger=INFO,dailyRollingFile

########## 按天拆分文件输出器 ##########
# 配置每天拆分文件输出器
log4j.appender.dailyRollingFile=org.apache.log4j.DailyRollingFileAppender
# 配置自定义格式器
log4j.appender.dailyRollingFile.layout=org.apache.log4j.PatternLayout
# 配置自定义转换模式
log4j.appender.dailyRollingFile.layout.conversionPattern=[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%-4rms] [%c] %M %L %m%n
# 配置文件路径及名称
log4j.appender.dailyRollingFile.file=../logDir/dailyRollingFile.log
# 配置字符编码
log4j.appender.dailyRollingFile.encoding=UTF-8
# 配置日期格式，默认按照天数拆分
#'.'yyyy-MM-dd HH:mm--按照分钟拆分
log4j.appender.dailyRollingFile.datePattern='.'yyyy-MM-dd
```
**保存日志到数据库**

相关配置：

导入依赖
```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.26</version>
</dependency>
```
创建表
```sql
create table log4j
(
    id         int    (11)    not null auto_increment comment '自增ID',
    name       varchar(30)    default null comment '项目名称',
    createTime varchar(30)    default null comment '创建时间',
    level      varchar(10)    default null comment '日志级别',
    thread     varchar(30)    default null comment '线程名称',
    className  varchar(255)   default null comment '全限定名',
    method     varchar(50)    default null comment '方法名称',
    lineNumber int    (5)     default null comment '代码行号',
    message    varchar(10000) default null comment '日志信息',
    primary key (id)
)
```
配置文件
```properties
#配置日志级别,输出控制器
log4j.rootLogger=INFO,db

########## 数据库输出器 ##########
#配置数据库输出器
log4j.appender.db=org.apache.log4j.jdbc.JDBCAppender
#配置自定义格式器
log4j.appender.db.layout=org.apache.log4j.PatternLayout
#配置数据库
log4j.appender.db.URL=jdbc:mysql://localhost:3306/test
log4j.appender.db.Driver=com.mysql.cj.jdbc.Driver
log4j.appender.db.User=root
log4j.appender.db.Password=root
log4j.appender.db.Sql=insert into log4j (name, createTime, level, thread, className, method, lineNumber, message) \
  values ('log', '%d{yyyy-MM-dd HH:mm:ss.SSS}', '%p', '%t', '%c', '%M', '%L', '%m');
```
### 3.6 自定义logger
之前创建的 logger 使用的都是继承 rootLogger，我们也可以自定义 logger，让其他 logger 来继承。这种继承关系是按照包结构来制定的。

在配置文件中使用`log4j.logger.`来指定父节点路径。如果根节点的 logger 和自定义父 logger 配置的**输出位置不同**，则二者都会生效，配置的位置都会输出。如果配置位置相同，则会重复记录两次。

但是如果**输出级别不同**，则按照自定义的父 logger 级别输出。
```java
// Log4jTest.class 位于 com.youyi.zhao 包下
Logger logger = Logger.getLogger(Log4jTest.class);
```
```properties
# 配置日志级别,输出控制器
log4j.rootLogger=INFO,console
# 自定义 logger
log4j.logger.com.youyi=INFO,file

########## 控制台输出器 ##########
# 配置控制台输出器
log4j.appender.console=org.apache.log4j.ConsoleAppender
# 配置自定义格式器
log4j.appender.console.layout=org.apache.log4j.PatternLayout
# 配置自定义转换模式
log4j.appender.console.layout.conversionPattern=[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%-4rms] [%c] %M %L %m%n

########## 文件输出器 ##########
# 配置文件输出器
# file 是自定义的输出器名称
log4j.appender.file=org.apache.log4j.FileAppender
# 配置自定义格式器
log4j.appender.file.layout=org.apache.log4j.PatternLayout
# 配置自定义转换模式
log4j.appender.file.layout.conversionPattern=[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%-4rms] [%c] %M %L %m%n
# 配置文件路径及名称
log4j.appender.file.file=../logDir/file.log
# 配置字符编码
log4j.appender.file.encoding=UTF-8
```
**应用场景**

我们之所以要自定义 logger，是为了针对不同包做更加灵活的操作。

例如，我们可以在原有案例基础上，增加一个 Apache 的 logger，这样当位于`org.apache`下的类发生错误时，会被记录到控制台上。
```properties
# 配置日志级别,输出控制器
log4j.rootLogger=INFO,console
# 自定义 logger
log4j.logger.com.youyi=INFO,file
# Apache
log4j.logger.org.apache=error,console

########## 控制台输出器 ##########
# 配置控制台输出器
log4j.appender.console=org.apache.log4j.ConsoleAppender
# 配置自定义格式器
log4j.appender.console.layout=org.apache.log4j.PatternLayout
# 配置自定义转换模式
log4j.appender.console.layout.conversionPattern=[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%-4rms] [%c] %M %L %m%n

########## 文件输出器 ##########
# 配置文件输出器
# file 是自定义的输出器名称
log4j.appender.file=org.apache.log4j.FileAppender
# 配置自定义格式器
log4j.appender.file.layout=org.apache.log4j.PatternLayout
# 配置自定义转换模式
log4j.appender.file.layout.conversionPattern=[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5p] [%t] [%-4rms] [%c] %M %L %m%n
# 配置文件路径及名称
log4j.appender.file.file=../logDir/file.log
# 配置字符编码
log4j.appender.file.encoding=UTF-8
```