## Spring Configuration Processor

>   它的作用和 主配置文件 application.properties 或者 application.yml 里面的 spring.profiles.active 有着相似的作用，但是不同的是，使用 spring.profiles.avtive，你添加的其他配置文件命名格式只能是 application-{name}.properties 或者 application-{name}.yml，而使用文件处理器这个依赖，则对文件名没有任何约束

### 1.pom依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

### 2.创建其他配置文件

```properties
love.you=aoxin
love.me=mdxl
love.they=78
```

- 这个配置文件先命名为 application-haha.properties，然后如果想要使用里面的属性值的时候，只能在application.propertiesz中加入

  ```properties
  spring.profiles.active=haha
  ```

- 这样，我们才能在代码类中获取到该值，如下

  ```java
  @RestController
  public class DockerController {
      @Value("${love.you}")
      private String name;
  
      @GetMapping("/test1")
      public String test1(){
          return name;
      }
  }
  ```

### 3.文件处理器

​    使用文件处理器，我们可以创建任意名字的配置文件，如 haha.properties,同时也不需要在application配置文件中引入，我们可以直接使用，不过有一个前提就是在引入它的属性值的类上，加上注解 @PropertySource("classpath:haha.properties") ，这样我们依旧可以使用，把之前active的引入删除

```java
@RestController
@PropertySource("classpath:haha.properties")
public class DockerController {

    @Value("${love.you}")
    private String name;

    @GetMapping("/test1")
    public String test1(){
        return name;
    }
}
```

### 4、为什么使用它？

​    那么看上去也并没有区别么，倒是有一点费劲的感觉，其实不然，因为有些配置文件里面的属性，有些开发工程师是直接想在配置类中使用，不想在主配置文件spring.profiles.active依赖，而这种属性值往往也不分开发环境、仿真环境和线上环境的，所以会有一小部分开发工程师乐意去使用它。

---

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

---

移除默认的 logback 日志

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

---

## 事务传播行为

用来描述这样一个现象：`methodA`开启了一个事务，调用了`methodB`，`methodB`是继续在`methodA`的事务中进行还是开启一个新的事物。

```java
@Transaction(Propagation=REQUIRED)
public void methodA() {
    methodB();
    //doSomething
}

@Transaction(Propagation=REQUIRED_NEW)
public void methodB() {
    //doSomething
}
```

**七种传播行为**

- `REQUIRED`（默认）

  如果当前有事务，则继续在事务中进行， 如果没有则开启一个新的事务

- `REQUIRED_NEW`

  无论是否存在事务，都会开启一个新的事务来执行，新老事务相互独立，一个事务抛出异常不会影响另一个事务的提交

- `NESTED`

  如果当前存在事务，则嵌套在当前事务中执行，如果没有事务则新建一个事务

  外部事务抛出异常会导致嵌套事务回滚，嵌套事务回滚不会影响外部事务

- `SUPPORTS`

  支持当前事务，如果当前不存在事务，就以非事务的方式进行

- `NOT_SUPPORTED`

  以非事务的方式执行，如果当前存在事务，则把当前事务挂起

- `MANDATORY`

  强制的事务执行，如果不存在事务就抛出异常

- `NEVER`

  以非事务的方式执行，如果存在事务则抛出异常

---

## spring、springMVC、springboot的区别

`spring`是一个 IOC 容器，用来管理 Bean，使用依赖注入实现控制反转，可以方便的整合各种框架，提供 AOP 机制弥补 OOP 的代码重复问题，更方便地将不同类不同方法中的共同处理抽取成切面，自动注入给方法执行，比如日志、异常等。

`springMVC`是`spring`对 web 框架的一个解决方案，提供了一个总的前端控制器`Servlet`，用来接受请求，然后定义了一套路由策略（url 到 handler 的映射）及适配执行 handler，将 handler 结果使用试图解析技术生成试图传递给前端。

`springboot`是`spring`提供的一个快速开发工具包，相当于`springMVC`的进阶版，让程序员能更方便、更快速的开发`spring+springMVC`应用，简化了配置（约定了默认配置，使用 class 代替 xml配置文件），整合了一系列的解决方案（starter 机制）。
