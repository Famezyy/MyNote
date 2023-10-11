# 第2章_JUL
JUL 全称 Java Util Logging，是 java 原生的框架（java.util.logging.Logger），不需引入第三方库，主要应用在小型项目中。
## 1.JUL组件
图1

Logger：记录器，应用程序获取 Logger 对象，调用其 API 发布日志信息，Logger 被认为是访问日志系统的入口程序。

Handler：处理器，每个 Logger 都会关联一个或一组 Handler，Logger 会将日志交给关联的 Handler 做处理，由 Handler 负责记录日志，handler 具体实现了日志的输出位置，比如可以输出到控制台或文件中。

Filter：过滤器，根据需要定制哪些信息会被记录。

Formatter：格式化组件，负责对日志中的数据和信息进行转换和格式化，决定了输出日志最终的形式。

Level：日志的输出级别，每条日志消息都有一个关联的级别。我们通过设置输出级别，来展现最终呈现的日志信息。

## 2.入门案例
```java
// 创建 Logger 对象，传入当前类的全路径字符串
Logger logger = Logger.getLogger("com.youyi.zhao.Tests");
// 1.调用日志级别相关的方法
logger.info("info");
// 2.调用通用的 log 方法，然后在里面定义 level 级别并设置输出信息的参数
logger.log(Level.INFO, "info");
// 3.占位符拼接字符串
String name = "zhao";
int age = 22;
logger.log(Level.INFO, "学生的姓名：{0}, 年龄:{1}", new Object[] {name, age});
```
结果
```bash
7 26, 2022 12:51:48 午後 com.youyi.zhao.Tests test
情報: info
7 26, 2022 12:51:48 午後 com.youyi.zhao.Tests test
情報: info
7 26, 2022 12:57:59 午後 com.youyi.zhao.Tests test
情報: 学生的姓名：zhao, 年龄:22
```
## 3.日志级别
### 3.1 默认日志级别
- SEVERE：错误 -- 最高级的日志级别
- WARNING：警告
- INFO：（默认级别）消息
- CONFIG：配置
- FINE：详细信息（少）
- FINER：详细信息（中）
- FINEST：详细信息（多）-- 最低级的日志级别

两个特殊的级别

- OFF：关闭日志记录
- ALL：启用所有消息的日志记录

对于日志的级别，我们重点关注的是 new 对象时的第二个参数，它是一个数值：
```bash
OFF Integer.MAX_VALUE

SEVERE 1000
WARNING 900

...
    
FINEST 300

ON Integer.MIN_VALUE
```
其意义在于，如果我们设置的日志级别是 INFO -- 800， 那么最终展现的日志信息，必须是数值大于等于 800 的所有日志信息，所以最终展现的就是：SEVERE、WARNING、INFO。
```java
// 只通过 setLevel 改变默认的级别不会起到效果，需要同时设置 handler 才会生效
logger.setLevel(Level.CONFIG);
// 默认只会输出比 info 级别高的信息： severe、warning、info
logger.severe("severe");
logger.warning("warning");
logger.info("info");
logger.config("config");
logger.fine("fine");
logger.finer("finer");
logger.finest("finest");
```
### 3.2 自定义日志级别
```java
// 将默认的日志打印方式关掉
// 设置 false，打印日志的方式就不会按照父 logger 默认的方式进行
logger.setUseParentHandlers(false);

// 处理器 handler
// 创建控制台日志处理器
ConsoleHandler handler = new ConsoleHandler();
// 创建日志格式化组件对象
SimpleFormatter formatter = new SimpleFormatter();
// 在处理其中设置输出格式
handler.setFormatter(formatter);
// 在记录其中添加处理器
logger.addHandler(handler);

// 设置打印级别
// 必须同时设置日志记录器和处理器的级别，最终取二者中的最小级别
logger.setLevel(Level.ALL);
handler.setLevel(Level.ALL);

logger.severe("severe");
logger.warning("warning");
logger.info("info");
logger.config("config");
logger.fine("fine");
logger.finer("finer");
logger.finest("finest");
```
## 4.常用设置
### 4.1 输出到文件中
相当于做了日志的持久化操作。
```java
// 将默认的日志打印方式关掉
// 设置 false，打印日志的方式就不会按照父 logger 默认的方式进行
logger.setUseParentHandlers(false);

// 创建文件日志处理器
FileHandler handler = new FileHandler("D:\\test\\logTest.log");
SimpleFormatter formatter = new SimpleFormatter();
handler.setFormatter(formatter);
logger.addHandler(handler);

logger.setLevel(Level.ALL);
handler.setLevel(Level.ALL);
```
### 4.2 添加多个处理器
```java
// 将默认的日志打印方式关掉
// 设置 false，打印日志的方式就不会按照父 logger 默认的方式进行
logger.setUseParentHandlers(false);

// 创建文件日志处理器
FileHandler handler = new FileHandler("D:\\test\\logTest.log");
SimpleFormatter formatter = new SimpleFormatter();
handler.setFormatter(formatter);
logger.addHandler(handler);

// 创建控制台处理器
ConsoleHandler handler2 = new ConsoleHandler();
handler2.setFormatter(formatter);
logger.addHandler(handler2);

logger.setLevel(Level.ALL);
handler.setLevel(Level.ALL);
```
用户使用 Logger 进行日志的记录，可设置多个 Handler，Logger 用于记录日志，Handler 用于输出日志。
### 4.3 Logger的父子关系
JUL 中 Logger 间存在父子关系，但是这种关系并不是继承关系，而是通过树状结构实现的。

JUL 在初始化时，会创建一个顶层的 RootLogger 作为所有 Logger 的父 Logger，它是 LoggerManager 的内部类，名称默认是一个空的字符串""。这个 RootLogger 对象作为树状结构的根节点存在，将来自定义的父子关系通过路径来关联。

```java
// logger1 的父引用为 RootLogger
Logger logger1 = Logger.getLogger("com.youyi");
// 可以认为 logger1 是 logger2 的父亲
Logger logger2 = Logger.getLogger("com.youyi.zhao");
Logger logger3 = Logger.getLogger("com.youyi.zhao.Tests");

System.out.println(logger2.getParent() == logger1); // true
System.out.println(logger3.getParent() == logger1); // false
```
父亲的设置通过能做用于儿子：
```java
Logger logger1 = Logger.getLogger("com.youyi");
Logger logger2 = Logger.getLogger("com.youyi.zhao");
Logger logger3 = Logger.getLogger("com.youyi.zhao.Tests");

// 改变父亲的级别
logger1.setUseParentHandlers(false);
ConsoleHandler handler = new ConsoleHandler();
SimpleFormatter formatter = new SimpleFormatter();
handler.setFormatter(formatter);
logger1.addHandler(handler);
handler.setLevel(Level.ALL);
logger1.setLevel(Level.ALL);

// 会影响到 logger2 和 logger3
logger3.severe("severe");
logger3.warning("warning");
logger3.info("info");
logger3.config("config");
logger3.fine("fine");
logger3.finer("finer");
logger3.finest("finest");
```
### 4.4 配置文件
如果没有添加配置文件，会使用默认的配置文件。默认会从`${java.home}.lib.logging.properties`读取配置。
```properties
# RootLogger 使用的处理器，在获取 RootLogger 对象时进行的设置
# 默认的情况下，下述配置的是控制台的处理器，只能往控制台上进行输出操作
# 添加新的处理器只需要通过","进行连接即可
handlers= java.util.logging.ConsoleHandler

# RootLogger 的日志级别
# 全局的日志级别，默认输出下述配置的级别
.level= INFO

# 文件处理器属性设置
# 输出日志文件的路径
# %h 指用户文件夹
java.util.logging.FileHandler.pattern = %h/java%u.log
# 输出日志文件的限制（50000字节）
java.util.logging.FileHandler.limit = 50000
# 输出日志文件的数量
java.util.logging.FileHandler.count = 1
# 输出日志的格式
# 默认 XML 的方式输出
java.util.logging.FileHandler.formatter = java.util.logging.XMLFormatter

# 控制台处理器属性设置
# 控制台默认 INFO 级别
java.util.logging.ConsoleHandler.level = INFO
# 控制台默认输出格式
java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter
# 也可以将日志级别设定到具体的包下
com.xyz.foo.level = SEVERE
```
读取自定义配置文件：
```java
// 将配置文件放在类路径下
InputStream is = this.getClass().getResourceAsStream("logging.properties");
// 取得日志管理器的对象
LogManager logManager = LogManager.getLogManager();
// 读取自定义配置文件
logManager.readConfiguration(is);

Logger logger = Logger.getLogger("com.youyi.zhao");

logger.severe("severe");
logger.warning("warning");
logger.info("info");
logger.config("config");
logger.fine("fine");
logger.finer("finer");
logger.finest("finest");
```
配置文件
```properties
handlers= java.util.logging.ConsoleHandler

.level= CONFIG

java.util.logging.FileHandler.pattern = C:/Users/APD-43164863/Documents/java%u.log
java.util.logging.FileHandler.limit = 50000
java.util.logging.FileHandler.count = 1
java.util.logging.FileHandler.formatter = java.util.logging.SimpleFormatter

# 自定义 Logger
com.youyi.handlers = java.util.logging.FileHandler
# 自定义日志等级
com.youyi.level = CONFIG
# 屏蔽父 Logger 的设置
com.youyi.useParentHandlers = false
```
注意，新日志会覆盖掉原来的日志，希望追加日志时，需配置：
```properties
java.util.logging.FileHandler.append=true
```
## 5.总结
原理解析：
1. 初始化 LogManagerLogManager：加载 logging.properties，添加 Logger 到 LogManager
2. 从单例的 LogManager 获取 Logger
3. 设置日志级别，在打印时使用 LogRecord 类
4. Filter 提供了日志级别外更细粒度的控制
5. Handler 决定了日志的输出位置，例如控制台、文件等
6. Formatter 用来格式化输出