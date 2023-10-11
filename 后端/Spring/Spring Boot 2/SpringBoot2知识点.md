## application/x-www-form-urlencoded 类型的请求

由 RequestParamMethodArgumentResolver 参数解析器解析

```java
HandlerMethodArgumentResolverComposite.class

private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
    HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
    if (result == null) {
        for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
            if (resolver.supportsParameter(parameter)) {
                result = resolver;
                this.argumentResolverCache.put(parameter, result);
                break;
            }
        }
    }
    return result;
}
```

|   参数注解    | Request | Map  | Pojo |
| :-----------: | :-----: | :--: | :--: |
|   不加注解    |    √    |  ×   |  √   |
| @RequestParam |    ×    |  √   |  ×   |
| @RequestBody  |    ×    |  ×   |  ×   |

## spring、springMVC、springboot的区别

`spring`是一个 IOC 容器，用来管理 Bean，使用依赖注入实现控制反转，可以方便的整合各种框架，提供 AOP 机制弥补 OOP 的代码重复问题，更方便地将不同类不同方法中的共同处理抽取成切面，自动注入给方法执行，比如日志、异常等。

> **扩展**
>
> IoC（Inversion of Control）：控制反转是一种软件设计原则，它将应用程序的控制权从应用程序代码转移到外部容器或框架。在传统的编程中，应用程序代码通常控制应用程序的流程，包括对象的创建和管理。在IoC中，这些控制权被反转，框架或容器负责创建、管理和提供对象，而应用程序代码只需使用这些对象。这种方式有助于降低代码的耦合性，提高代码的可维护性和可测试性。

`springMVC`是`spring`对 web 框架的一个解决方案，提供了一个总的前端控制器`Servlet`，用来接受请求，然后定义了一套路由策略（url 到 handler 的映射）及适配执行 handler，将 handler 结果使用试图解析技术生成试图传递给前端。

`springboot`是`spring`提供的一个快速开发工具包，相当于`springMVC`的进阶版，让程序员能更方便、更快速的开发`spring+springMVC`应用，简化了配置（约定了默认配置，使用 class 代替 xml 配置文件），整合了一系列的解决方案（starter 机制）。

## IOC

`IOC容器`，实际上就是个`map(key, value)`，里面存的是`各种对象`（在 xml 里配置的 bean 节点、@Repository、@Service、@Controller、@Component），在项目启动的时候会读取配置文件里面的 bean 节点，根据全限定类名使用反射创建对象放到 map 里。

这个时候 map 里就有各种对象了，接下来我们在代码里需要用到对象时，再通过`DI注入`（@Autowired、@Resource 等注解，以及 xml 里 bean 节点的 ref 属性）

**控制反转**

没有引入 IOC 容器之前，对象 A 依赖于对象 B，那么对象 A 在初始化或者运行到某一点的时候，自己必须主动去创建对象 B 或者使用已经创建的对象 B，无论是创建还是使用对象 B，控制全都在自己手上。

引入 IOC 容器后，对象 A 与对象 B 之间失去了直接联系，当对象 A 运行到需要对象 B 的时候，IOC 容器会主动创建一个对象 B 注入到对象 A 需要的地方。

可以看出，对象 A 获得依赖对象 B 的过程由主动行为变成了被动行为，控制权颠倒过来了，这就是`控制反转`的由来。

全部对象的控制权全部上缴给`IOC容器`，所以`IOC容器`成了整个系统的关键核心，他起到了粘合剂的作用，把系统中的所有对象粘合在一起发挥作用，如果没有这个粘合剂，对象之间就彼此失去了联系。

**依赖注入**

获得依赖对象的过程由自身管理变成了由`IOC容器`主动注入，依赖注入是实现`IOC`的方法，就是由`IOC容器`在运行期间，动态地将某种依赖关系注入到对象之中。

## 一个注解，优雅的实现循环重试功能

在实际工作中，重处理是一个非常常见的场景，比如：

- 发送消息失败。
- 调用远程服务失败。
- 争抢锁失败。

这些错误可能是因为网络波动造成的，等待过后重处理就能成功。通常来说，会用`try/catch`，`while`循环之类的语法来进行重处理，但是这样的做法缺乏统一性，并且不是很方便，要多写很多代码。然而`spring-retry`却可以通过注解，在不入侵原有业务逻辑代码的方式下，优雅的实现重处理功能。

### 1.@Retryable是什么？

spring 系列的`spring-retry`是另一个实用程序模块，可以帮助我们以标准方式处理任何特定操作的重试。在`spring-retry`中，所有配置都是基于简单注释的。

### 2.使用步骤

#### POM依赖

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

#### 启用`@Retryable`

```java
@EnableRetry
@SpringBootApplication
public class HelloApplication {
    public static void main(String[] args) {
        SpringApplication.run(HelloApplication.class, args);
    }
}
```

#### 在方法上添加`@Retryable`

```java
@Service
public class TestRetryServiceImpl implements TestRetryService {
 
    @Override
    @Retryable(value = Exception.class, maxAttempts = 3, backoff = @Backoff(delay = 2000,multiplier = 1.5))
    public int test(int code) throws Exception{
        System.out.println("test被调用,时间："+LocalTime.now());
          if (code==0){
              throw new Exception("情况不对头！");
          }
        System.out.println("test被调用,情况对头了！");
 
        return 200;
    }
}
```

来简单解释一下注解中几个参数的含义：

- `value`：抛出指定异常才会重试
- `include`：和 value 一样，默认为空，当 exclude 也为空时，默认所有异常
- `exclude`：指定不处理的异常
- `maxAttempts`：最大重试次数，默认3次
- `backoff`：重试等待策略，默认使用`@Backoff`，`@Backoff`的 value 默认为1000L，我们设置为2000L；`multiplier`（指定延迟倍数）默认为 0，表示固定暂停 1 秒后进行重试，如果把`multiplier`设置为1.5，则第一次重试为 2 秒，第二次为 3 秒，第三次为 4.5 秒。

**当重试耗尽时还是失败，会出现什么情况呢？**

当重试耗尽时，`RetryOperations`可以将控制传递给另一个回调，即`RecoveryCallback`。`Spring-Retry`还提供了`@Recover`注解，用于 @Retryable 重试失败后处理方法。如果不需要回调方法，可以直接不写回调方法，那么实现的效果是，重试次数完了后，如果还是没成功没符合业务判断，就抛出异常。

#### @Recover

```java
@Recover
public int recover(Exception e, int code){
   System.out.println("回调方法执行！！！！");
   //记日志到数据库 或者调用其余的方法
    return 400;
}
```

可以看到传参里面写的是 `Exception e`，这个是作为回调的接头暗号（重试次数用完了，还是失败，我们抛出这个`Exception e`通知触发这个回调方法）。对于`@Recover`注解的方法，需要特别注意的是：

- 方法的返回值必须与`@Retryable`方法一致
- 方法的第一个参数，必须是 Throwable 类型的，建议是与`@Retryable`配置的异常一致，其他的参数，需要哪个参数，写进去就可以了（`@Recover`方法中有的）
- 该回调方法与重试方法写在同一个实现类里面

#### 注意事项

- 由于是基于 AOP 实现，所以不支持类里自调用方法
- 如果重试失败需要给`@Recover`注解的方法做后续处理，那这个重试的方法不能有返回值，只能是 void
- 方法内不能使用`try catch`，只能往外抛异常
- `@Recover`注解来开启重试失败后调用的方法(注意，需跟重处理方法在同一个类中)，此注解注释的方法参数一定要是`@Retryable`抛出的异常，否则无法识别，可以在该方法中进行日志处理。

## starter

### 1.简介

为了让开发者在开发 spring 生态下的企业级应用的时候，只需要关心业务逻辑，减少对配置和外部环境的依赖。

主要作用有以下几个：

- 维护对应 jar 包的版本依赖，开发者可以不需要关心版本冲突等容易出错的细节

- 自动导入对应功能的所有的 jar 包依赖

- 导入 starter 后，这个组件会自动集成到 Spring 生态中，对于相关的 bean 的管理也是基于自动装配机制来完成的

- 依赖 starter 组件后，这些功能所需要维护的外部化配置会自动集成到 Spring Boot 中，只需要在 application.properties 中配置

  > 导入`xxxAutoConfiguration` -- 导入`xxx`组件 -- 绑定`xxxProperties` -- 绑定配置文件项
  >
  > 例：
  >
  > ```java
  > @AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
  > @Configuration(proxyBeanMethods = false)
  > @ConditionalOnWebApplication(type = Type.SERVLET)
  > @ConditionalOnClass(DispatcherServlet.class)
  > @AutoConfigureAfter(ServletWebServerFactoryAutoConfiguration.class)
  > public class DispatcherServletAutoConfiguration {
  > 
  >     @Configuration(proxyBeanMethods = false)
  >     @Conditional(DefaultDispatcherServletCondition.class)
  >     @ConditionalOnClass(ServletRegistration.class)
  >     @EnableConfigurationProperties(WebMvcProperties.class)
  >     protected static class DispatcherServletConfiguration {
  >         @Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
  >         public DispatcherServlet dispatcherServlet(WebMvcProperties webMvcProperties) {
  >             DispatcherServlet dispatcherServlet = new DispatcherServlet();
  >             dispatcherServlet.setDispatchOptionsRequest(webMvcProperties.isDispatchOptionsRequest());
  >             dispatcherServlet.setDispatchTraceRequest(webMvcProperties.isDispatchTraceRequest());
  >             dispatcherServlet.setThrowExceptionIfNoHandlerFound(webMvcProperties.isThrowExceptionIfNoHandlerFound());
  >             dispatcherServlet.setPublishEvents(webMvcProperties.isPublishRequestHandledEvents());
  >             dispatcherServlet.setEnableLoggingRequestDetails(webMvcProperties.isLogRequestDetails());
  >             return dispatcherServlet;
  >         }
  >     }
  > }
  > 
  > @ConfigurationProperties(prefix = "spring.mvc")
  > public class WebMvcProperties {
  > }
  > ```

> 命名方式
>
> - 官方维护的 starter：spring-boot-starter-xxx
> - 第三方维护的 starter：xxx-spring-boot-starter

### 2.启动原理

`starter -- autoconfigure -- spring-boot-starter`

autoconfigure 包中配置使用`META-INF/spring.factories`中`EnableAutoConfiguration`的值，使得项目启动加载指定的自动配置类。

> 在 3.0 版本后发生了改变：在`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`文件中配置。

### 3.自定义starter

1. `youyi-zhao-spring-boot-starter-autoconfigure`：自动配置包

   **编写逻辑**

   ```java
   @Configuration
   @EnableConfigurationProperties(HelloProperties.class)
   public class HelloAutoConfiguration {
   
       @ConditionalOnMissingBean(HelloUtil.class)
       @Bean
       public HelloUtil helloUtil(HelloProperties helloProperties) {
           HelloUtil helloUtil = new HelloUtil();
           helloUtil.setPrefix(helloProperties.getPrefix());
           helloUtil.setSuffix(helloProperties.getSuffix());
           return helloUtil;
       }
   
   }
   
   @ConfigurationProperties("youyi.zhao.hello")
   @Data
   @AllArgsConstructor
   @NoArgsConstructor
   public class HelloProperties {
       private String prefix;
       private String suffix;
   }
   
   @Data
   @AllArgsConstructor
   @NoArgsConstructor
   public class HelloUtil {
       private String prefix;
       private String suffix;
   
       public String sayHello(String username) {
           return prefix + ":<" + username + ">" + suffix;
       }
   }
   ```

   **在 sources/META-INF 下新建一个 spring.factories **

   ```properties
   org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.youyi.hello.HelloAutoConfiguration
   ```

   **配置 pom 文件**

   ```xml
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>
   
       <groupId>com.youyi.hello</groupId>
       <artifactId>youyi-zhao-spring-boot-starter-autoconfigure</artifactId>
       <version>1.0-SNAPSHOT</version>
   
       <properties>
           <maven.compiler.source>8</maven.compiler.source>
           <maven.compiler.target>8</maven.compiler.target>
       </properties>
   
       <dependencies>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter</artifactId>
               <version>2.7.0</version>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-configuration-processor</artifactId>
               <version>2.7.0</version>
           </dependency>
           <dependency>
               <groupId>org.projectlombok</groupId>
               <artifactId>lombok</artifactId>
               <version>RELEASE</version>
               <scope>compile</scope>
           </dependency>
       </dependencies>
   
       <build>
           <plugins>
               <plugin>
                   <groupId>org.springframework.boot</groupId>
                   <artifactId>spring-boot-maven-plugin</artifactId>
                   <version>2.7.0</version>
               </plugin>
           </plugins>
       </build>
   
   </project>
   ```

   构成如下：

   <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220626204240534-8753ddc31363e4d3dde82a575d868621-c58c99.png" alt="image-20220626204240534" style="zoom:80%;" />

2. `youyi-zhao-spring-boot-starter`：启动器

   仅放入一个 pom 文件：

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>
   
       <groupId>com.youyi.hello</groupId>
       <artifactId>youyi-zhao-spring-boot-starter</artifactId>
       <version>1.0-SNAPSHOT</version>
   
       <properties>
           <maven.compiler.source>8</maven.compiler.source>
           <maven.compiler.target>8</maven.compiler.target>
           <youyi-zhao-spring-boot-starter-autoconfigure>1.0-SNAPSHOT</youyi-zhao-spring-boot-starter-autoconfigure>
       </properties>
   
       <!-- 指定 autoconfigure 包 -->
       <dependencies>
           <dependency>
               <groupId>com.youyi.hello</groupId>
               <artifactId>youyi-zhao-spring-boot-starter-autoconfigure</artifactId>
               <version>${youyi-zhao-spring-boot-starter-autoconfigure}</version>
           </dependency>
       </dependencies>
   
   </project>
   ```

接下来在自己的包中导入 starter 即可：

```xml
<dependency>
    <groupId>com.youyi.hello</groupId>
    <artifactId>youyi-zhao-spring-boot-starter</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

### 4.配置元数据的应用

在使用Spring Boot开发应用的时候，你是否有发现这样的情况：自定义属性是有高量背景的，鼠标放上去，有一个`Cannot resolve configuration property`的配置警告。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/pasted-416-371e44563206b051351c6ac70fda9396-799edc.png" alt="img" style="zoom: 33%;" />

如果不对于这个警告觉得烦，想要去掉，那么可以通过设置来去除：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/pasted-417-490d99a60c44d9b9190b3815c8c70ea8-c7fe3c.png" alt="img" style="zoom: 33%;" />

但是，我的建议是不要去掉，因为这个警告正好可以通过高亮来区分你的自定义配置以及框架配置，可以让你快速的分辨哪些是自定义的。

如果你实在想去掉，那么也不建议用上面说的方法，而是建议通过完善配置元数据的方式来完成。所以，今天就来具体说说配置元数据的应用！

#### 啥是配置元数据？

我们不妨打开一个已经创建好的 Spring Boot 项目，查看一下它的 Spring Boot 依赖包，可以找到如下图的一个 json 文件：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/pasted-418-576a784db9b54e15ba83f01bf55ac6fd-e8112d.png" alt="img" style="zoom: 33%;" />

这里报错的就是配置的元数据信息。有没有发现这些`name`的值都很熟悉？其中`description`是不是也很熟悉？对，这些就是我们常用的 Spring Boot 原生配置的元数据信息。

这下知道配置元数据可以用来做啥了吧？它可以帮助 IDE 来完成配置联想和配置提示的展示。

而我们自定义配置之所以会报警告，同时也没有提示信息，就是因为没有这个元数据的配置文件！

#### 配置元数据的自动生成

既然知道了原理，那么接下来我们尝试用一下配置元数据试试！

**第一步**：创建一个配置类，定义一个自定义配置

```java
@Data
@Configuration
@ConfigurationProperties(prefix = "com.didispace")
public class DidiProperties {
    
    /**
     * 这是一个测试配置
     */
    private String from;

}
```

**第二步**：在`pom.xml`中添加自动生成配置元数据的依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
</dependency>
```

**第三步**：`mvn install`下这个项目。

此时我们可以在工程target目录下找到元数据文件：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/pasted-419-0bac7d2b4294ac0ad586cf0e71c65ab2-bbe30c.png" alt="img" style="zoom: 33%;" />

同时，我们在配置文件中尝试编写这个自定义的配置项时，可以看到编译器给出了联想和提示：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/pasted-420-c244f2fe484077daedd274207c09b51b-76694f.png" alt="img" style="zoom: 33%;" />

并且，编写完配置之后，也没有高亮警告了！

## 定时任务

### 1.简单定时任务

对于一些比较简单的定时任务，比如固定时间间隔执行固定方法，在标准 Java 方法上注解`@Scheduled`即可

```java
@Component
public class ScheduledTask {

    @Scheduled(cron = "0/10 * * * * ?") //每10秒执行一次
    public void scheduledTaskByCorn() {
        LoggerUtils.info("定时任务开始 ByCorn：" + DateUtils.dateFormat());
        scheduledTask();
        LoggerUtils.info("定时任务结束 ByCorn：" + DateUtils.dateFormat());
    }

    @Scheduled(fixedRate = 10000) //每10秒执行一次
    public void scheduledTaskByFixedRate() {
        LoggerUtils.info("定时任务开始 ByFixedRate：" + DateUtils.dateFormat());
        scheduledTask();
        LoggerUtils.info("定时任务结束 ByFixedRate：" + DateUtils.dateFormat());
    }

    @Scheduled(fixedDelay = 10000) //每10秒执行一次
    public void scheduledTaskByFixedDelay() {
        LoggerUtils.info("定时任务开始 ByFixedDelay：" + DateUtils.dateFormat());
        scheduledTask();
        LoggerUtils.info("定时任务结束 ByFixedDelay：" + DateUtils.dateFormat());
    }

    private void scheduledTask() {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

然后项目启动类上增加注解`@EnableScheduling`，表示开启定时任务

```java
@SpringBootApplication
@EnableScheduling
public class SpringBootDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootDemoApplication.class, args);
    }
}
```

这里因为我们在 ScheduledTask 类创建了三个定时任务，@Scheduled 默认是不并发执行的，因此我们先注释掉其他，分别进行测试。

> `@Scheduled(cron = "0/10 * * ?")`控制的每 10 秒执行一次的定时任务，是每 10 秒整执行一次，即一分钟内，如果当前秒数能够整除 10，则执行定时任务，即只会在10s，20s，30s...的时候执行，如果配置定时任务`@Scheduled(cron = "0/7 * * ?")`这种，则只会在0s，7s，14s...的时候执行。
>
> 对于`fixedRate`来说，如果业务代码执行时间小于定时任务间隔时间，那么定时任务每 10 秒执行一次，且不受业务代码影响，无论业务代码执行多久，定时任务都是 10 秒执行一次；如果业务代码执行时间大于定时任务间隔时间，则定时任务循环执行。
>
> 对于`fixedDelay`来说，不管业务代码执行时间与定时任务间隔时间熟长熟短，定时任务都会等业务代码执行完成后再开启新一轮定时。

### 2.corn表达式

corn 表达式格式：秒 分 时 日 月 星期 年（可选）

| 字段名     | 允许的值        | 允许的特殊字符   |
| ---------- | --------------- | ---------------- |
| 秒         | 0-59            | , - * /          |
| 分         | 0-59            | , - * /          |
| 时         | 0-23            | , - * /          |
| 日         | 1-31            | , - * ? / L W C  |
| 月         | 1-12 or JAN-DEC | , - * /          |
| 星期       | 1-7 or SUN-SAT  | , - *  ? / L C # |
| 年（可选） | 1970-2099       | , - * /          |

*****：通配符，表示该字段可以接收任意值。

**？** ：表示不确定的值，或不关心它为何值，仅在日期和星期中使用，当其中一个设置了条件时，另外一个用"?" 来表示"任何值"。

**,**：表示多个值，附加一个生效的值。

**-**：表示一个指定的范围

**/**：指定一个值的增量值。例n/m表示从n开始，每次增加m

**L**：用在日期表示当月的最后一天，用在星期"L"单独使用时就等于"7"或"SAT"，如果和数字联合使用表示该月最后一个星期X。例如，"0L"表示该月最后一个星期日。

**W**：指定离给定日期最近的工作日（周一到周五），可以用"LW"表示该月最后一个工作日。例如，"10W"表示这个月离10号最近的工作日

**C**：表示和calendar联系后计算过的值。例如：用在日期中，"5C"表示该月第5天或之后包括calendar的第一天；用在星期中，"5C"表示这周四或之后包括calendar的第 一天。

**#**：表示该月第几个星期X。例6#3表示该月第三个周五。

**示例值**

0 * *** ? 每分钟触发

0 0 * * * ? 每小时整触发

0 0 4 * * ? 每天凌晨4点触发

0 15 10 * * ? 每天早上10：15触发

*/5 ** * * ? 每隔5秒触发

0 */5 ** * ? 每隔5分钟触发

0 0 4 1 * ? 每月1号凌晨4点触发

0 0 4 L * ? 每月最后一天凌晨3点触发

0 0 3 ? * L 每周星期六凌晨3点触发

0 11,22,33 * * * ? 每小时11分、22分、33分触发

### 3.配置定时任务

对于上面那些简单的定时任务，定时任务的 corn 表达式写死在代码里，如果要改动表达式，需要修改代码，重新打包发布，比较麻烦。因此，我们可以把 corn 表达式配置在配置文件中，然后程序读取配置，当需要修改表达式时，只需要修改配置文件即可。

application.yml 增加配置

```yaml
demo:
  corn: 0/11 * * * * ?
```

定时任务

```java
@Component
public class ScheduledTask {

    @Scheduled(cron = "${demo.corn}")
    public void scheduledTaskByConfig() {
        LoggerUtils.info("定时任务 ByConfig：" + DateUtils.dateFormat());
    }
}
```

### 4.动态修改定时任务

对于有些情况，我们需要在代码中，通过方法动态修改定时任务corn表达式

application.yml 配置

```yaml
demo:
  corn: 0/7 * * * * ?
  cornV2: 0/22 * * * * ?
```

新建ScheduledTaskV2.java

```java
@Component
public class ScheduledTaskV2 implements SchedulingConfigurer {

    @Value("${demo.corn}")
    private String corn;
    @Value("${demo.cornV2}")
    private String cornV2;

    private int tag = 0;

    @Override
    public void configureTasks(ScheduledTaskRegistrar scheduledTaskRegistrar) {
        scheduledTaskRegistrar.addTriggerTask(() -> {
            LoggerUtils.info("定时任务V2：" + DateUtils.dateFormat());
        }, (triggerContext) -> {
            CronTrigger cronTrigger;
            if (tag % 2 == 0) {
                LoggerUtils.info("定时任务V2动态修改corn表达式：" + corn + "," + DateUtils.dateFormat());
                cronTrigger = new CronTrigger(corn);
                tag++;
            } else {
                LoggerUtils.info("定时任务V2动态修改corn表达式：" + cornV2 + "," + DateUtils.dateFormat());
                cronTrigger = new CronTrigger(cornV2);
                tag++;
            }

            return cronTrigger.nextExecutionTime(triggerContext);
        });
    }
}
```

### 5.并发执行定时任务

定时任务类添加注解`@EnableAsync`，需并发执行的定时任务方法添加注解`@Async`

```java
@Component
@EnableAsync
public class ScheduledTaskV3 {

    @Scheduled(cron = "0/7 * * * * ?")
    @Async
    public void scheduledTaskV1() {
        LoggerUtils.info("定时任务V3，定时任务1开始：" + DateUtils.dateFormat());
        scheduledTask();
        LoggerUtils.info("定时任务V3，定时任务1结束：" + DateUtils.dateFormat());
    }

    @Scheduled(cron = "0/10 * * * * ?")
    @Async
    public void scheduledTaskV2() {
        LoggerUtils.info("定时任务V3，定时任务2开始：" + DateUtils.dateFormat());
        scheduledTask();
        LoggerUtils.info("定时任务V3，定时任务2结束：" + DateUtils.dateFormat());
    }

    @Scheduled(cron = "0/22 * * * * ?")
    @Async
    public void scheduledTaskV3() {
        LoggerUtils.info("定时任务V3，定时任务3开始：" + DateUtils.dateFormat());
        scheduledTask();
        LoggerUtils.info("定时任务V3，定时任务3结束：" + DateUtils.dateFormat());
    }

    private void scheduledTask() {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

## 热部署

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
</dependency>
```

项目或者页面修改以后：`ctrl+F9`

## 兼容Junit4及之前版本

SpringBoot 2.2.0 版本开始后引入 Junit5 作为单元测试默认库，要想兼容 Junit4，需要引入一下依赖：

```xml
<dependency>
	<groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <scope>test</scope>
    <exclusions>
    	<exclusion>
        	<groupId>org.hamcrest</groupId>
            <artifactId>hamcrest-core</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

## 读取Resources目录

这是一个公共方法，用来读取文件中的内容的方法。

```java
public static void printFileContent(Object obj) throws IOException {
    if (null == obj) {
        throw new RuntimeException("参数为空");
    }
    BufferedReader reader = null;
    // 如果是文件路径
    if (obj instanceof String) {
        reader = new BufferedReader(new FileReader(new File((String) obj)));
    // 如果是文件输入流
    } else if (obj instanceof InputStream) {
        reader = new BufferedReader(new InputStreamReader((InputStream) obj));
    }
    String line = null;
    while ((line = reader.readLine()) != null) {
        System.out.println(line);
    }
    reader.close();
}
```

- `T.class.getClassLoader().getResourceAsStream()`

  此方法默认是从 classpath 路径（即：src 或 resources 路径下）下查找文件的，所以，路径前不需要加 “/”。

  ```java
  public class ResourceUtil {
  
      public void getResource(String fileName) throws IOException{
          InputStream in = this.getClass().getClassLoader().getResourceAsStream(fileName);
          printFileContent(in);
      }
  
      public static void main(String[] args) throws IOException {
          new ResourceUtil().getResource("config/test.properties");
      }
  
  }
  ```

- `T.class..getResourceAsStream()`

  此方法跟要读取的文件与当前.class 文件的位置有关。如果 test.properties 和 ResourceUtil 在同一个文件夹下，那么：`this.getClass().getResourceAsStream(“test.properties”)`，
  如果 test.properties 和 ResourceUtil 不在同一个文件夹下，那么：`this.getClass().getResourceAsStream(“/config/test.properties”)`。

  ```java
  public void getResource2(String fileName) throws IOException{
      InputStream in = this.getClass().getResourceAsStream("/" + fileName);
      printFileContent(in);
  }
  
  public static void main(String[] args) throws IOException {
      new ResourceUtil().getResource2("config/test.properties");
  }
  ```

- `ClassPathResource`

  ```java
  public void getResource3(String fileName) throws IOException{
      ClassPathResource classPathResource = new ClassPathResource(fileName);
      printFileContent(classPathResource.getInputStream());
  }
  
  public static void main(String[] args) throws IOException {
      new ResourceUtil().getResource3("config/test.properties");
  }
  ```

  path 前加不加 “/” 无所谓。即使是一个 jar 包，也依旧能读取到。

- `PathMatchingResourcePatternResolver`

  ```java
  private static final PathMatchingResourcePatternResolver RESOLVER = new PathMatchingResourcePatternResolver();
  
  public static Resource resolveConfigLocation(String configLocation) {
      return RESOLVER.getResource(configLocation);
  }
  
  /**
   * 以相对文件路径的方式读取{@code resource}资源目录下的文件资源
   *
   * @param fileLocation 文件相对路径 比方说{@code resource}下的"file.txt"文件 那么参数便为"file.txt"
   * @return 文件的字符内容
   */
  public static String readFileContent(String fileLocation) {
      try (InputStream is = resolveConfigLocation(fileLocation).getInputStream()) {
          StringWriter sw = new StringWriter();
          IOUtils.copy(is, sw, StandardCharsets.UTF_8);
          return sw.toString();
      } catch (IOException e) {
          throw new RuntimeException(String.format("相对路径{%s}文件读取失败", fileLocation));
      }
  }
  ```


## 异步执行

- 在配置类上开启异步

  ```java
  @EnableAsync
  @Configuration
  public class Configuration(){}
  ```

- 配置线程池

  默认异步任务使用的一个线程池的 corePoolSize = 8, 阻塞队列采用了无界队列 LinkedBlockingQueue，会造成内存溢出。

  - 通过配置文件

    ```properties
    spring.task.execution.pool.core-size=5
    spring.task.execution.pool.max-size=50
    spring.task.execution.pool.queue-capacity=200
    spring.task.execution.thread-name-prefix=asyncTask
    ```

  - 通过配置类

    ```java
    @EnableAsync
    @Configuration
    public class AsyncConfiguration() implements AsyncConfigurer {
        @Bean(name="defaultTaskExecutor")
        public ThreadPoolTaskExecutor defaultExecutor() {
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            executor.setCorePoolSize(5);
            executor.setMaxPoolSize(50);
            executor.setQueueCapacity(200);
            executor.setKeepAliveSeconds(200);
            executor.setThreadNamePrefix("asyncTask");
            executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
            executor.initialize();
            return executor;
        }
        
        @Bean(name="otherTaskExecutor")
        public ThreadPoolTaskExecutor otherExecutor() {
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            ...
            return executor;
        }
        
        // 指定默认线程池
        @Override
        public Executor getAsyncExecutor() {
            return defaultExecutor();
        }
        
        
    }
    ```

- 在方法上使用`@Async`

  ```java
  @Async
  public CompletableFuture<POJO> handle() {
      return CompletableFuture.completedFuture(pOJO);
  }
  ```

  可以指定使用的线程池

  ```java
  @Async("defaultTaskExecutor")
  public CompletableFuture<POJO> handle1() {
      return CompletableFuture.completedFuture(pOJO);
  }
  
  @Async("otherTaskExecutor")
  public CompletableFuture<POJO> handle2() {
      return CompletableFuture.completedFuture(pOJO);
  }
  ```

- 同步阻塞等待结果

  ```java
  CompletableFuture.allof(completableFuture1, completableFuture2);
  pOJO1 = completableFuture1.get();
  pOJO2 = completableFuture2.get();
  ```


## 跨域问题

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CoreRegistry registry) {
        registry.addMapping("/**")
            .allowedOrigins("http://localhost:9090")
            .allowedMethods("*")
            .allowedHeaders("header1", "header2", "header3")
            .exposedHeaders("header1", "header2")
            .allowCredentials(false).maxAge(3600);
    }
}
```

## 连接数

最大连接数为`max-connections`和`accept-count`的和，当再有连接进来时不会直接返回而是先等待，超过超时时间后返回超时错误

```yaml
server:
	tomcat:
		threads:
			# 最小线程数
			min-spare: 10
			# 最大线程数
			max: 20
        # 最大连接数，默认 8192，需要 1024 倍数
        max-connections: 30
        # 最大等待数，默认 100
        accept-count: 10
```

