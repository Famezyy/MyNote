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

