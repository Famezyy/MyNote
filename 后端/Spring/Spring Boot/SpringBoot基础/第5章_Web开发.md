# 第5章_Web开发

## 1.简介

常见的配置项：

- Spring MVC 相关：`spring.mvc`
- Web 场景通用配置：`spring.web`
- 文件上传配置：`spring.servlet.multipart`
- 服务器的配置：`server`

常见的配置：

- 包含了 `ContentNegotiatingViewResolver` 和 `BeanNameViewResolver` 组件，方便视图解析
- 默认的静态资源处理机制：静态资源放在 `static` 文件夹下即可访问
- 自动注册了 `Converter`、`GenericConverter`、`Formatter` 组件，适配常见的数据类型转换和格式化需求
- 提供 `HttpMessageConverters`，支持返回 `json` 等数据类型
- 提供 `MessageCodesResolver`，方便国际化及错误消息处理
- 支持静态 `index.html`
- 提供 `ConfigurableWebBindingInitializer`，实现消息处理、数据绑定、类型转化等功能

## 2.静态资源访问

### 2.1 基础使用

**（1）静态资源目录**

静态资源放在类路径下：`classpath:/static`、`classpath:/public`、`classpath:/resources`、`classpath:/META-INF/resources`。

访问时需要访问：当前项目根路径 + / + 静态资源名。

> **原理**
>
> 收到请求时，先去找对应的 `Controller` 看谁能处理。找不到的所有请求都交给静态资源处理器。静态资源也找不到时则相应 404 页面。

**（2）改变默认的静态资源路径**

```yaml
# 静态资源访问路径（设置该值会导致 index.html 和 Favicon 不能被访问，因为 controll 默认只处理 /index 请求）
spring:
  mvc:
    static-path-pattern: /res/**
# 静态资源存放路径
  web:
    resources:
      static-locations: classpath:/haha/
```

**（3）自定义浏览器图标**

将 `favicon.ico` 放在静态资源目录下即可。

**（4）修改启动图**

```yaml
spring:
  banner:
    image:
      location: classpath:2.png
```

### 2.2 静态资源配置原理

`WebMvcAutoConfiguration` 的内部静态类 `WebMvcAutoConfigurationAdapter`，其实现了 `WebMvcConfigurer`，重写了 `addResourceHandlers` 方法配置了静态资源的处理规则。

```java
public class WebMvcAutoConfiguration {
    
    @Import({WebMvcAutoConfiguration.EnableWebMvcConfiguration.class})
    @EnableConfigurationProperties({ WebMvcProperties.class, WebProperties.class })
    @Order(0)
    public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer, ServletContextAware {
        
        @Override
		public void addResourceHandlers(ResourceHandlerRegistry registry) {
            // spring.web.resources.add-mappings：是否禁用所有静态资源访问规则
			if (!this.resourceProperties.isAddMappings()) {
				logger.debug("Default resource handling disabled");
				return;
			}
			addResourceHandler(registry, "/webjars/**", "classpath:/META-INF/resources/webjars/");
            // staticPathPattern = "/**"
			addResourceHandler(registry, this.mvcProperties.getStaticPathPattern(), (registration) -> {
                // CLASSPATH_RESOURCE_LOCATIONS = { "classpath:/META-INF/resources/", "classpath:/resources/", "classpath:/static/", "classpath:/public/" };
				registration.addResourceLocations(this.resourceProperties.getStaticLocations());
				if (this.servletContext != null) {
					ServletContextResource resource = new ServletContextResource(this.servletContext, SERVLET_LOCATION);
					registration.addResourceLocations(resource);
				}
			});
		}
        
         private void addResourceHandler(ResourceHandlerRegistry registry, String pattern, Consumer<ResourceHandlerRegistration> customizer) {
            if (!registry.hasMappingForPattern(pattern)) {
                ResourceHandlerRegistration registration = registry.addResourceHandler(new String[]{pattern});
                customizer.accept(registration);
                // 设置缓存周期，表示多久不用找服务器更新，默认没有
                registration.setCachePeriod(this.getSeconds(this.resourceProperties.getCache().getPeriod()));
                // HTTP 缓存控制
                registration.setCacheControl(this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl());
                // 是否使用最后一次修改
                registration.setUseLastModified(this.resourceProperties.getCache().isUseLastModified());
                this.customizeResourceHandlerRegistration(registration);
            }
        }
        
    }
    
}
```

**规则**

- 访问 `/webjars/**` 就去 `classpath:/META-INF/resources/webjars/` 找

- 访问 `staticPathPattern` 就去 `CLASSPATH_RESOURCE_LOCATIONS` 找

- 静态资源默认都有缓存规则，缓存设置通过 `spring.web` 来配置


### 2.3 欢迎页的处理规则

`WebMvcAutoConfiguration` 导入了 `EnableWebMvcConfiguration`，它创建了 `WelcomePageHandlerMapping` 来处理欢迎页。

```java
@EnableConfigurationProperties({WebProperties.class})
public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration implements ResourceLoaderAware {
    @Bean
    public WelcomePageHandlerMapping welcomePageHandlerMapping(ApplicationContext applicationContext, FormattingConversionService mvcConversionService, ResourceUrlProvider mvcResourceUrlProvider) {
        // 创建 WelcomePageHandlerMapping
        return (WelcomePageHandlerMapping)this.createWelcomePageHandlerMapping(applicationContext, mvcConversionService, mvcResourceUrlProvider, WelcomePageHandlerMapping::new);
    }
}
```

```java
final class WelcomePageHandlerMapping extends AbstractUrlHandlerMapping {
    WelcomePageHandlerMapping(TemplateAvailabilityProviders templateAvailabilityProviders, ApplicationContext applicationContext, Resource indexHtmlResource, String staticPathPattern) {
        this.setOrder(2);
        // 创建 WelcomePage
        WelcomePage welcomePage = WelcomePage.resolve(templateAvailabilityProviders, applicationContext, indexHtmlResource, staticPathPattern);
        if (welcomePage != WelcomePage.UNRESOLVED) {
            logger.info(LogMessage.of(() -> {
                return !welcomePage.isTemplated() ? "Adding welcome page: " + indexHtmlResource : "Adding welcome page template: index";
            }));
            ParameterizableViewController controller = new ParameterizableViewController();
            controller.setViewName(welcomePage.getViewName());
            this.setRootHandler(controller);
        }
    }
}
```

```java
final class WelcomePage {
    static WelcomePage resolve(TemplateAvailabilityProviders templateAvailabilityProviders, ApplicationContext applicationContext, Resource indexHtmlResource, String staticPathPattern) {
        if (indexHtmlResource != null && "/**".equals(staticPathPattern)) {
            return new WelcomePage("forward:index.html", false);
        } else {
            return templateAvailabilityProviders.getProvider("index", applicationContext) != null ? new WelcomePage("index", true) : UNRESOLVED;
        }
    }
}
```

默认访问 `/**` 时会重定向到 `index.html`。

### 2.4 自定义静态资源配置

#### 1.通过配置文件

```properties
# 指定静态资源访问路径前缀，会覆盖
spring.mvc.static-path-pattern=/static/**
# 指定静态资源存放目录，会覆盖
spring.web.resources.static-locations=classpath:/a/,classpath:/b/
# 设置缓存时间
spring.web.resources.cache.period=3600
# 缓存详细合并项控制
# 设置缓存时间，时间范围内浏览器无需从服务器再次获取；会覆盖上面的配置
spring.web.resources.cache.cachecontrol.max-age=7200
# 使用 last-modified 属性来判断资源是否经过修改，默认开启
spring.web.resources.cache.use-last-modified=true
```

#### 2.通过配置类WebMvcConfigurer

```java
@Configuration
public class MyConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        WebMvcConfigurer.super.addResourceHandlers(registry);
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/a/", "classpath:/b/")
                .setCacheControl(CacheControl.maxAge(7200, TimeUnit.SECONDS));
    }
}
```

## 3.数据响应与内容协商

### 3.1 数据响应

#### 1.SpringMVC支持的返回值类型

- ModelAndView

- Model

- View

- ResponseEntity

  ```java
  public ResponseEntity<Person> getPersonInfo() throws InterruptedException {
      return new ResponseEntity<>(new Person(), HttpStatus.OK);
  }
  ```

- ResponseBodyEmitter

- StreamingResponseBody

- HttpEntity

- HttpHeaders

- Callable

- DeferredResult

- ListenableFuture

- CompletionStage

- WebAsyncTask

- 标注 `@ModelAttribute` 注解且为对象类型

- 标注 `@ResponseBody` 注解：最终会由 `MessageConverter` 来处理

#### 2.默认的MessageConverter支持类型

在 `WebMvcConfigurationSupport` 的 `addDefaultHttpMessageConverters` 中会判断系统中是否加载了相应的类，从而加载默认的 `MessageConverter`。

- ByteArrayHttpMessageConverter：Byte
- StringHttpMessageConverter：String（UTF-8）
- StringHttpMessageConverter：String（ISO-8859-1）
- ResourceRegionHttpMessageConverter：ResourceRegion
- ResourceSourceHttpMessageConverter：Resource
- AllEncompassingFormHttpMessageConverter：MultiValueMap
- MappingJackson2HttpMessageConverter：所有类型
- MappingJackson2HttpMessageConverter：所有类型
- Jaxb2RootElementHttpMessageConverter：注解方式 xml 处理的（只有导入了 xml 包才会添加）

#### 3.响应JSON底层

- 执行完 handler 方法后，会调用返回值处理器的 `supportsReturnType()` 方法查找支持的返回值处理器

- 当标注了 `@ResponseBody` 时，由`RequestResponseBodyMethodProcessor` 来解析返回值

- 然后遍历所有的 `MessageConverter` 的 `canWrite()` 方法，最终由 `AbstractJackson2HttpMessageConverter` 来负责写出操作

- 底层利用 jackon 的 objectMapper 转换

### 3.2 内容协商

获得浏览器传入的接受的媒体类型，返回对应的媒体类型。

#### 1.开启支持xml内容协商

SpringBoot 默认只集成了 JSON 场景，需要手动引入 XML 场景依赖：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

#### 2.为实体类添加xml注解

```java
@Data
@JacksonXmlRootElement
public class Person {
    ...
}
```

#### 3.开启浏览器参数方式内容协商

浏览器默认使用基于请求头的策略 `HeaderContentNegotiationStrategy`，会基于请求头 `headers` 中的 `ACCEPT` 参数进行判断。

也可以基于请求参数进行判断，但需要添加参数方式策略 `ParameterContentNegotiationStrategy`，但该策略仅支持 `xml` 和 `json`。

```yaml
spring:
	mvc:
    	contentnegotiation:
      		favor-parameter: true  # 会添加基于参数的内容协商策略
      		parameter-name: type   # 修改参数名，默认是 format
```

此时发送请求：`http://localhost:8080/test/person?type=json` 或者 `http://localhost:8080/test/person?type=xml` 即可。

### 3.3 原理

- 执行 `handler()` 方法处理返回值时，会先获得浏览器传入的接受的媒体类型。默认使用基于请求头的内容协商策略，会获得 `headers` 中的 `ACCEPT` 参数
- 然后获取可产生的媒体类型，遍历所有 `messageConverter`，调用相应的 `converter.canWrite()` 方法查询支持写入相应的类型的 `converter`
- 匹配接受的媒体类型和产生的媒体类型，选择最合适的 `mediaType`
- 循环遍历所有 `messageConverters` 查找支持相应类型转换的 `Converters`
- 调用对应转换器的 `write()` 方法完成写入

### 3.4 自定义MessageConverter

**方法1**

直接向容器中注册一个 `AbstractHttpMessageConverter` 类型的组件：

```xml
<!-- 引入 yaml 依赖 -->
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-yaml</artifactId>
</dependency>
```

```java
@Component
public class YamlMessageConverter extends AbstractHttpMessageConverter<Object> {

    private ObjectMapper objectMapper;

    protected YamlMessageConverter() {
        super(new MediaType("application", "yaml", StandardCharsets.UTF_8));
        // 传入一个 yaml 工厂，同时禁止在开头输出 ---
        this.objectMapper = new ObjectMapper(new YAMLFactory().disable(YAMLGenerator.Feature.WRITE_DOC_START_MARKER));
    }

    @Override
    protected boolean supports(Class<?> clazz) {
        return true;
    }

    // @RequestBody
    @Override
    protected Object readInternal(Class<?> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        return null;
    }

    @Override
    protected void writeInternal(Object o, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        try (OutputStream ops = outputMessage.getBody()) {
            objectMapper.writeValue(ops, o);
        }
    }

}
```

> **注意**
>
> 直接实现 `HttpMessageConverter` 时需要注意 `canWrite()` 在 `MediaType=null` 时要返回 true。

**方法2**

不使用 `@Component` 注解，在 `WebMvcConfigurer` 添加：

```java
@Configuration
public class MyMapperConfiguration implements WebMvcConfigurer {

    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new YamlMessageConverter());
    }
}
```

**设置支持基于参数的内容协商**

需要使自定义的 `MessageConverter` 支持基于参数的内容协商时，需要配置内容协商策略。因为 `ParameterContentNegotiationStrategy` 策略仅支持 `xml` 和 `json`。

```java
@Configuration
public class MyMapperConfiguration implements WebMvcConfigurer {

    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        Map<String, MediaType> mediaTypes = new HashMap<>();
        mediaTypes.put("json", MediaType.APPLICATION_JSON);
        mediaTypes.put("xml", MediaType.APPLICATION_XML);
        mediaTypes.put("youyi", MediaType.parseMediaType("application/youyi"));
        ParameterContentNegotiationStrategy parameterContentNegotiationStrategy = new ParameterContentNegotiationStrategy(mediaTypes);
        // 设置请求参数名（默认format）
        parameterContentNegotiationStrategy.setParameterName("mediaType");
        configurer.strategies(Collections.singletonList(parameterContentNegotiationStrategy));
    }
}
```

但此时会覆盖原有的内容协商管理器，**使基于请求头的策略也失效**。此时若 mediaType 无法匹配，会默认为全匹配 `*/*`，此时会与 `json` 的格式匹配上。

可以在新添加一个基于请求头的策略：

```java
@Configuration
public class MyMapperConfiguration implements WebMvcConfigurer {

    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        Map<String, MediaType> mediaTypes = new HashMap<>();
        mediaTypes.put("json", MediaType.APPLICATION_JSON);
        mediaTypes.put("xml", MediaType.APPLICATION_XML);
        mediaTypes.put("youyi", MediaType.parseMediaType("application/youyi"));
        ParameterContentNegotiationStrategy parameterContentNegotiationStrategy = new ParameterContentNegotiationStrategy(mediaTypes);
        parameterContentNegotiationStrategy.setParameterName("mediaType");
        // 新添加一个 HeaderContentNegotiationStrategy
        HeaderContentNegotiationStrategy headerContentNegotiationStrategy = new HeaderContentNegotiationStrategy();
        configurer.strategies(Arrays.asList(parameterContentNegotiationStrategy, headerContentNegotiationStrategy));
    }
}
```

### 3.5 ResponseBodyAdvice

允许在执行`@ResponseBody`或`ResponseEntity`控制器方法之后但在使用`HttpMessageConverter`编写正文之前自定义响应。

实现可以直接使用`RequestMappingHandlerAdapter`和`ExceptionHandlerExceptionResolver`进行注册，或者使用`@ControllerAdvice`进行注释，在这种情况下它们将被两者自动检测到。

可用于记录 log。

```java
@ControllerAdvice
public class MyResponseBodyAdvice implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return !returnType.getParameterType().isAssignableFrom(Bservice.class);
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        if (returnType.getGenericParameterType().equals(String.class)) {
            ObjectMapper objectMapper = new ObjectMapper();
            try {
                return objectMapper.writeValueAsString(new Bservice());
            } catch (JsonProcessingException e) {
                e.printStackTrace();
            }
        }
        return new Bservice();
    }
}
```

### 3.6 内容定位

底层就是通过 Bean 名称从容器中获取相应的 Bean。

```java
public class client {
    
    @Autowired
    ParserFactory partserFactory;
    
    public String getAll(ContentType type) {
        switch (type) {
            case CSV:
                return csvParser.parse(type);
                break;
            case JSON:
                return jsonParser.parse(type);
                break;
        }
        return parserFactory.getParser(type).parse(reader);
    }
}

@Configuration
public class ParserConfig {
    
    @Bean("parserFactory")
    public FactoryBean serviceLocatorFactoryBean() {
        ServiceLocatorFactoryBean factoryBean = new ServiceLocatorFactoryBean();
        // 传入服务定位接口，底层会创建动态代理，根据方法的第一个参数值（调用了 toString，所以使用 enum 也可以）获取 Bean 并返回
        factoryBean.setServiceLocatorInterface(ParserFactory.class);
        return factotyBean;
    }
    
}

public interface ParserFactory {
    Parser getParser(ContentType type);
}

@Component("CSV")
public class CSVParser implements Parser {
    public String parse(Reader r) {...}
}
```

## 4.视图解析与模板引擎

如果不是一个 `RestController` 则返回 String 时会自动将其视为视图并进行解析，如果无法解析则抛出相关错误。

### 4.1 视图解析

`SpringBoot` 默认不支持 JSP，需要引入第三方模块引擎技术实现页面渲染。

**原理流程**

- 在处理派发结果时，会调用 `DispatcherServlet `的 `render()` 方法进行页面渲染（出错或者返回值为 JSON 时，mv 为 null，不会调用该方法）

- 根据 view 名称创建 view 对象，一般由 `ContentNegotiationViewResolver` 解析

  - error：`ErrorMvcAutoconfiguration$StaticView`

  - `forward:/**`：`InternalResourceView`

    > `request.getRequestDispatcher(path).forward(request, response)`

  - `redirect:/**`：`RedirectView`

    > `response.sendRedirect(encodedURL)`

  - string：**默认没有解析器**，可添加 thymeleaf 作为解析器

- 调用对应 view 的`render()`方法进行渲染

### 4.2 模板解析-thymeleaf

Thymeleaf is a modern server-side Java template engine for both web and standalone environments, capable of processing HTML, XML, JavaScript, CSS and even plain text.

#### 1.基本语法

##### 1.1 表达式

| 表达式名字 | 语法                                                         | 用途                                                         |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 变量取值   | `${...}`                                                     | 获取请求域、session域、对象                                  |
| 选择变量   | `*{...}`                                                     | 获取上下文对象值                                             |
| 消息       | `#{...}`                                                     | 获取国际化等值                                               |
| 连接       | `@{...}`：自动加上当前项目路径<br />`@{/...}`：自动加上访问路径 | 生成链接，例如`th:src="@{/static/1.jpg}"`、`th:src="@{${url}}"` |
| 片段表达式 | `~{...}`                                                     | `jsp.include` 作用，引入公共页面                             |

##### 1.2 字面量

文本值用单引号：`'text'`

数字：1

布尔值：true，false

控制：null

变量：x，y（变量名不能有空格）

##### 1.3 字符串拼接

- `"${'前缀' + name + '后缀'}"`
- `"|前缀 ${name} 后缀|"`

##### 1.4 数字运算

+、-、*、/、%

##### 1.5 布尔运算

运算符：and、or

##### 1.6 比较运算

- 比较：>、<、<=、>=（gt、lt、ge、le）
- 等值运算：==、!=（eq、ne）

##### 1.7 条件运算

- (if) ? (then)
- (if) ? (then) : (else)
- (value) ? (defaultvalue)

##### 1.8行内使用

```html
<img src="1.png" />
[[${session.loginUser.username}]]
<span class="create"></span>
```

##### 1.9变量选择

```html
<div th:object="${session.user}">
    <p>Name: <span th:text="*{firstName}">Sebastian</span>.</p>
    <p>Surname: <span th:text="*{lastName}">Pepper</span>.</p>
    <p>Nationality: <span th:text="*{nationality}">Saturn</span>.</p>
</div>
```

等同于

```html
<div>
    <p>Name: <span th:text="${session.user.firstName}">Sebastian</span>.</p>
    <p>Surname: <span th:text="${session.user.lastName}">Pepper</span>.</p>
    <p>Nationality: <span th:text="${session.user.nationality}">Saturn</span>.</p>
</div
```

#### 2.标签属性内容

**（1）设置标签属性值-th:attr**

```html
<form action="subscribe.html" th:attr="action=@{/subscribe}">
    <fieldset>
        <input type="text" name="email" />
        <input type="submit" value="Subscribe!" th:attr="value=#{subscribe.submit}" />
    </fieldset>
</form>
```

设置多个值

```html
<img src="1.png" th:attr="src=@{2.png}, title=#{logo}" />
```

**替代写法：th:属性值名称**

```html
<form action="subscribe.html" th:action="@{/subscribe}">
<input type="submit" value="Subscribe!" th:value="#{subscribe.submit}" />
```

**（2）替换标签体内容-th:tex**

- `<h1 th:text="{msg}"></>`：会转义 `msg` 中的 html 标签

- `<h1 th:utext="{msg}"></>`：不会转移 html 标签，会原样显示

#### 3.迭代

```html
<tr th:each="prod : ${prods}" >
	<td th:text="${prod.name}">Onions</td>
    <td th:text="${prod.price}">2.41</td>
    <td th:text="${prod.inStock} ? #{true} : #{false}">yes</td>
</tr>

<tr th:each="prod,iterStat : ${prods}" th:class="${iterStat.odd} ? 'odd'">
	<td th:text="${prod.name}">Onions</td>
    <td th:text="${prod.price}">2.41</td>
    <td th:text="${prod.inStock} ? #{true} : #{false}">yes</td>
</tr>
```

`iterStat` 有以下属性：

- `index`：当前遍历元素的索引，从 0 开始
- `count`：当前遍历元素的索引，从 1 开始
- `size`：需要遍历元素的总数量
- `current`：当前正在遍历的元素对象，调用 `toString` 方法打印
- `even/odd`：是否偶数/奇数行
- `first`：是否第一个元素
- `last`：是否最后一个元素

#### 4.条件运算

```html
<a href="comments.html"
   th.href="@{/product/comments(prodId=${prod.id})}"
   th:if="${not #lists.isEmpty(prod.comments)}">view</a>

<div th:switch="${user.role}">
    <p th:case="'admin'">
        User is an administrator
    </p>
    <p th:case="#{roles.manager}">
		User is a manager        
    </p>
    <p th:case="*">
        User is some other thing
    </p>
</div>
```

#### 5.属性优先级

| Order | Feature                         | Attributes                                       |
| ----- | ------------------------------- | ------------------------------------------------ |
| 1     | Fragment inclusion              | th:insert<br />th:replace                        |
| 2     | Fragment iteration              | th:each                                          |
| 3     | Conditional Evaluation          | th:if<br />th:unless<br />th:switch<br />th:case |
| 4     | LocalVariable Definition        | th:object<br />th:with                           |
| 5     | General attribute modification  | th:attr<br />th:attrprepend<br />th:attrappend   |
| 6     | Specific attribute modification | th:value<br />th:href<br />th:src<br />...       |
| 7     | Text (tag body modification)    | th:text<br />th:utext                            |
| 8     | Fragment specification          | th:fragment                                      |
| 9     | Fragment removal                | th:remove                                        |

#### 6.内置工具

[**详细文档**](https://www.thymeleaf.org/doc/tutorials/3.1/usingthymeleaf.html#appendix-a-expression-basic-objects)

- `param`：请求参数对象

- `session`：session 对象

- `application`：application 对象

- `#execInfo`：模板执行信息

- `#messages`：国际化消息

- `#uris`：uri/url 工具

- `#conversions`：类型转换工具

- `#dates`：日期工具，是 `java.util.Date` 对象的工具类

- `#calendars`：类似 #dates，只不过是 `java.util.Calendar` 对象的工具类

- `#temporals`： JDK8+ `java.time` API 工具类

- `#numbers`：数字操作工具

- `#strings`：字符串操作

  ```html
  <!-- 转大写 -->
  <h1 th:text="${#string.toUpperCase(name)}"></h1>
  ```

- `#objects`：对象操作

- `#bools`：bool 操作

- `#arrays`：array 工具

- `#lists`：list 工具

- `#sets`：set 工具

- `#maps`：map 工具

- `#aggregates`：集合聚合工具（sum、avg）

- `#ids`：id 生成工具

#### 7.模板布局

- 定义模板： `th:fragment`
- 引用模板：`~{templateName::selector}`
- 插入模板：`th:insert`、`th:replace`

```html
<footer th:fragment="copy">&copy; 2011 The Good Thymes Virtual Grocery</footer>

<body>
    <div th:insert="~{footer :: copy}"></div>
    <div th:replace="~{footer :: copy}"></div>
</body>
```

结果：

```html
<body>
    <div>
        <footer>&copy; 2011 The Good Thymes Virtual Grocery</footer>
    </div>
    <footer>&copy; 2011 The Good Thymes Virtual Grocery</footer>
</body>
```

### 4.3 thymeleaf使用

#### 1.引入Starter

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

会自动导入组件

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(ThymeleafProperties.class)
@ConditionalOnClass({TemplateMode.class, SpringTemplateEngine.class})
@AutoConfigureAfter({WebMvcAutoConfiguration.class, WebFluxAutoConfiguration.class})
public class ThymeleafAutoConfiguration{}
```

自动配置好的策略：

- 所有 thymeleaf 的配置值都在 ThymeleafProperties
- 配置好了 SpringTemplateEngine
- 配置好了 ThymeleafViewResolver
- 所有的模板页面在 `classpath:/templates/`
- 后缀为 `.html`

#### 2.页面开发

```html
<body>
    <h1 th:text="${msg}">
        test
    </h1>
    <h2>
    	<a href="www.baidu.com" th:href="${link}">baidu</a><br/>
        <a href="www.baidu.com" th:href="@{link}">baidu</a><br/>
    </h2>
</body>
```

#### 3.后台管理

```java
@PostMapping("/login")
public String main(User user, HttpSession session, Model model) {
    if (StringUtils.hasLength(user.getUserName()) && "123".equals(user.getPassword())) {
        // 把登陆成功的用户保存下来
        session.setAttribute("loginUser", user);
        return "redirect:/main.html";
    } else {
        model.addAttribute("msg", "incorrect password");
        // thymeleaf 会试图在 /resources/templates 下找 login.html 模板文件
        return "login";
    }
}
```

### 4.配置

```properties
spring.thymeleaf.prefix=classpath:/templates/
spring.thymeleaf.suffix=.html
# 是否使用缓存，默认 true，开发环境可以关闭
spring.thymeleaf.cache=true
# 检查模板是否存在，默认 true，生产环境可以关闭
spring.thymeleaf.check-template=false
```

## 5.拦截器

### 5.1 自定义拦截器

```java
public class MyInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String requestURI = request.getRequestURI();
        HttpSession session = request.getSession();
        Object loginUser = session.getAttribute("loginUser");
        if(loginUser != null) {
            // 放行
            return true;
        }

        request.setAttribute("msg", "please login first");
        request.getRequestDispatcher("/").forward(request, response);
        return false;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
    }
}
```

配置拦截器

```java
@Configuration
public class MyMapperConfiguration implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new MyInterceptor())
            // “/*” 表示单层匹配，"/**"表示全路径匹配
                .addPathPatterns("/**")
                .excludePathPatterns("/", "/login", "/css/**", "/fonts/**", "images/**", "js/*");
    }
}
```

### 5.2 拦截器执行流程

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220621183514475-fd7e9ce762be13ab6a1c113b5d526fe3-47a1ee.png" alt="image-20220621183514475" style="zoom:80%;" />

<img src="https://img-blog.csdnimg.cn/d27c00135d3d400e8166d90ac9cc2240.png" style="zoom: 50%;" >

如果某个拦截器返回 false，`postHandle`方法都不会被执行，另外只有返回 true 的拦截器的`afterCompletion`方法才会执行。

<img src="https://img-blog.csdnimg.cn/a50f3b4ae85e4a6b8e4d54405ad6701a.png" style="zoom:50%;" >

### 5.3 源码

一个请求打进来时，DispatcherServlet 中的 doDispatch 方法会进行拦截器的调用，如下代码注释部分：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        try {
            ModelAndView mv = null;
            Object dispatchException = null;

            try {
                processedRequest = this.checkMultipart(request);
                multipartRequestParsed = processedRequest != request;
                mappedHandler = this.getHandler(processedRequest);
                if (mappedHandler == null) {
                    this.noHandlerFound(processedRequest, response);
                    return;
                }

                HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler());
                String method = request.getMethod();
                boolean isGet = "GET".equals(method);
                if (isGet || "HEAD".equals(method)) {
                    long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                    if ((new ServletWebRequest(request, response)).checkNotModified(lastModified) && isGet) {
                        return;
                    }
                }
                // 执行拦截器的全部 preHandle 方法
                if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                    return;
                }

                mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
                if (asyncManager.isConcurrentHandlingStarted()) {
                    return;
                }

                this.applyDefaultViewName(processedRequest, mv);
                // 执行拦截器的全部 postHandle 方法
                mappedHandler.applyPostHandle(processedRequest, response, mv);
            } catch (Exception var20) {
                dispatchException = var20;
            } catch (Throwable var21) {
                dispatchException = new NestedServletException("Handler dispatch failed", var21);
            }

            // 执行拦截器的全部 afterCompletion 方法
            this.processDispatchResult(processedRequest, response, mappedHandler, mv, (Exception)dispatchException);
        } catch (Exception var22) {
            // 异常后，执行拦截器的全部 afterCompletion 方法
            this.triggerAfterCompletion(processedRequest, response, mappedHandler, var22);
        } catch (Throwable var23) {
            // 异常后，执行拦截器的全部 afterCompletion 方法
            this.triggerAfterCompletion(processedRequest, response, mappedHandler, new NestedServletException("Handler processing failed", var23));
        }

    } finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            if (mappedHandler != null) {
mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        } else if (multipartRequestParsed) {
            this.cleanupMultipart(processedRequest);
        }
    }
}
```

在`mappedHandler.applyPreHandle()`方法中，通过`HandlerInterceptor[] interceptors = this.getInterceptors()`依序获得`registry.addInterceptor()`中加入的拦截器，for 循环正序执行拦截器逻辑：

```java
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HandlerInterceptor[] interceptors = this.getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        for(int i = 0; i < interceptors.length; this.interceptorIndex = i++) {
            HandlerInterceptor interceptor = interceptors[i];
            if (!interceptor.preHandle(request, response, this.handler)) {
                this.triggerAfterCompletion(request, response, (Exception)null);
                return false;
            }
        }
    }
    return true;
}
```

而在`mappedHandler.applyPostHandle()`方法中，同样是通过`HandlerInterceptor[] interceptors = this.getInterceptors()`依序获得`registry.addInterceptor()`中加入的拦截器，不过在 for 循环中是倒序执行逻辑：
```java
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv) throws Exception {
    HandlerInterceptor[] interceptors = this.getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        for(int i = interceptors.length - 1; i >= 0; --i) {
            HandlerInterceptor interceptor = interceptors[i];
            interceptor.postHandle(request, response, this.handler, mv);
        }
    }
}
```

最后看一下`mappedHandler.triggerAfterCompletion()`方法，同样 for 循环里是倒序执行逻辑：

```java
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex) throws Exception {
    HandlerInterceptor[] interceptors = this.getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        for(int i = this.interceptorIndex; i >= 0; --i) {
            HandlerInterceptor interceptor = interceptors[i];

            try {
                interceptor.afterCompletion(request, response, this.handler, ex);
            } catch (Throwable var8) {
                logger.error("HandlerInterceptor.afterCompletion threw exception", var8);
            }
        }
    }
}
```

上面的拦截器都是按照我们配置中的顺序执行，那如果我们想要自定义顺序该怎么办呢，`InterceptorRegistry`类的`addInterceptor`和`getInterceptors`方法源码如下，我们可以看到`addInterceptor`是将拦截器加入 list 集合，在`getInterceptors`方法中`this.registrations.stream().sorted(INTERCEPTOR_ORDER_COMPARATOR)`，拦截器 list 集合按照 order 升序排序，默认 order 为 0，可以在配置中用`.order(int order)`设置顺序。

```java
public class InterceptorRegistry {
    private final List<InterceptorRegistration> registrations = new ArrayList();
    private static final Comparator<Object> INTERCEPTOR_ORDER_COMPARATOR;
 
    public InterceptorRegistry() {
    }
 
    public InterceptorRegistration addInterceptor(HandlerInterceptor interceptor) {
        InterceptorRegistration registration = new InterceptorRegistration(interceptor);
        this.registrations.add(registration);
        return registration;
    }
 
    public InterceptorRegistration addWebRequestInterceptor(WebRequestInterceptor interceptor) {
        WebRequestHandlerInterceptorAdapter adapted = new WebRequestHandlerInterceptorAdapter(interceptor);
        InterceptorRegistration registration = new InterceptorRegistration(adapted);
        this.registrations.add(registration);
        return registration;
    }
 
    protected List<Object> getInterceptors() {
        return (List)this.registrations.stream().sorted(INTERCEPTOR_ORDER_COMPARATOR).map(InterceptorRegistration::getInterceptor).collect(Collectors.toList());
    }
 
    static {
        INTERCEPTOR_ORDER_COMPARATOR = OrderComparator.INSTANCE.withSourceProvider((object) -> {
            if (object instanceof InterceptorRegistration) {
                InterceptorRegistration var10000 = (InterceptorRegistration)object;
                ((InterceptorRegistration)object).getClass();
                return var10000::getOrder;
            } else {
                return null;
            }
        });
    }
}
```

### 5.4 拦截器与过滤器区别

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20221126145109515-e114f94e27ce22b901c7f0927a429035-150191.png" alt="image-20221126145109515" style="zoom:67%;" />

1. 过滤器和拦截器触发时机不一样，过滤器是在请求进入容器后，但请求进入 servlet 之前进行预处理的；拦截器是在进入 controller 前进行预处理
2. filter 接口在`javax.servlet`包下面，interceptor 在`org.springframework.web.servlet`下面，可以使用 spring 的组件
3. filter 通过`dochain`放行，intercetor 通过`prehandler`放行，并且粒度更细，还有`posthandler`、`aftercompletion`方法来处理 controller 执行后或者异常发生后的逻辑

## 6.文件上传

### 6.1 页面表单

```html
<form method="post" action="/upload" enctype="multypart/form-data">
    <input type="file" name="file"><br>
    <input type="submit" value="submit">
</form>
```

### 6.2 后端服务

```java
@PostMapping("/upload")
public String upload(@RequestPart("headerImg") MultipartFile headerImg, @RequestPart("photos") MultipartFile[] photos) throws IOException {
    if (!headerImg.isEmpty()) {
        String originalFilename = headerImg.getOriginalFilename();
        headerImg.transferTo(new File("H:\\cache\\" + originalFilename));
    }
    if (photos.length > 0) {
        for (MultipartFile photo : photos) {
            String originalFilename = photo.getOriginalFilename();
            photo.transferTo(new File("H:\\cache\\" + originalFilename));
        }
    }
    return "success";
}
```

### 6.3 自动配置原理

文件上传自动配置类`MultipartAutoConfiguration-MultipartProperties`自动配置了`StandardServletMultipartResolver`。

- 当收到请求后，DispatcherServlet 会首先判断是否是 multipart 请求（文件上传请求），调用`StandardServletMultipartResolver`的`isMultipart()`

  - 如果是，则封装 request 为`MultipartHttpServletRequest`

- 解析参数的值时，由`RequestPartMethodArgumentResolver`来处理，并将参数封装为`MultipartFile`

- 调用`transferTo()`方法时，底层调用`StreamUtils`的`copy()`

  ```java
  public abstract class StreamUtils {
      public static int copy(InputStream in, OutputStream out) throws IOException {
          Assert.notNull(in, "No InputStream specified");
          Assert.notNull(out, "No OutputStream specified");
  
          int byteCount = 0;
          byte[] buffer = new byte[BUFFER_SIZE];
          int bytesRead;
          while ((bytesRead = in.read(buffer)) != -1) {
              out.write(buffer, 0, bytesRead);
              byteCount += bytesRead;
          }
          out.flush();
          return byteCount;
      }
  }
  ```

## 8.原生组件注入

导入 web 原生组件，例如：Servlet、Filter、Listener。

### 8.1 使用注解

在配置类上标注：

```java
@ServletComponentScan(basePackages="com.youyi.zhao") // 指定原生 Servlet 组件存放包
```

在原生组件上标注：

```java
@WebServlet(urlPatterns="/my") // 不会经过 Spring 的拦截器
public class MyServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        resp.getWriter().write("test");
    }
}

@WebFilter(urlPatterns={"/css/*", "/images/*"})
public class MyFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) {}
    
    @Override
    public void doFilter(ServletRequest res, ServletResponse resp, FilterChain chain) {
        chain.doFilter(request, response);
    }
    
    @Override
    public void destroy() {}
}

@WebListener
```

### 8.2 使用RegistrationBean

```java
@Configuration
public class MyRegistConfig {
    @Bean
    public ServletRegistrationBean myServlet() {
        MyServlet myServlet = new MyServlet();
        return new ServletRegistrationBean(myServlet, "/my", "/my02");
    }
    
    @Bean
    public FilterRegistrationBean myFilter() {
        MyFilter myFilter = new MyFilter();
        // return new FIlterRegistrationBean(myFilter, myServlet()); 会拦截 myServlet 的访问路径
        FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean(myFilter);
        // “/*” 是一层匹配，“/**” 是全路径匹配
        filterRegistrationBean.setUrlPatterns(Arrays.asList("/my", "/css/*"));
        return filterRegistrationBean;
    }
    
    @Bean
    public ServletListenerRegistrationBean myListener() {
        MyServletContextListener myServletContextListener = new MyServletContextListener();
        return new ServletListenerRegistrationBean(myServletContextListener);
    }
}
```

### 8.3 扩展：DispatchServlet的注册

通过`DispatcherServletRegistrationBean`配置`DispatchServlet`的访问路径为`/`。

```java
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(DispatcherServlet.class)
@AutoConfigureAfter(ServletWebServerFactoryAutoConfiguration.class)
public class DispatcherServletAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Conditional(DefaultDispatcherServletCondition.class)
    @ConditionalOnClass(ServletRegistration.class)
    @EnableConfigurationProperties(WebMvcProperties.class)
    protected static class DispatcherServletConfiguration {

        // 向容器中放入一个 DispatcherServlet
        @Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
        public DispatcherServlet dispatcherServlet(WebMvcProperties webMvcProperties) {
            DispatcherServlet dispatcherServlet = new DispatcherServlet();
            dispatcherServlet.setDispatchOptionsRequest(webMvcProperties.isDispatchOptionsRequest());
            dispatcherServlet.setDispatchTraceRequest(webMvcProperties.isDispatchTraceRequest());
            dispatcherServlet.setThrowExceptionIfNoHandlerFound(webMvcProperties.isThrowExceptionIfNoHandlerFound());
            dispatcherServlet.setPublishEvents(webMvcProperties.isPublishRequestHandledEvents());
            dispatcherServlet.setEnableLoggingRequestDetails(webMvcProperties.isLogRequestDetails());
            return dispatcherServlet;
        }
    }

    @Configuration(proxyBeanMethods = false)
    @Conditional(DispatcherServletRegistrationCondition.class)
    @ConditionalOnClass(ServletRegistration.class)
    @EnableConfigurationProperties(WebMvcProperties.class)
    @Import(DispatcherServletConfiguration.class)
    protected static class DispatcherServletRegistrationConfiguration {

        // 配置 DispatcherServlet
        @Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
        @ConditionalOnBean(value = DispatcherServlet.class, name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
        public DispatcherServletRegistrationBean dispatcherServletRegistration(DispatcherServlet dispatcherServlet,
                                                                               WebMvcProperties webMvcProperties, ObjectProvider<MultipartConfigElement> multipartConfig) {
            // webMvcProperties.getServlet().getPath() 默认为 /
            DispatcherServletRegistrationBean registration = new DispatcherServletRegistrationBean(dispatcherServlet,
                                                                                                   webMvcProperties.getServlet().getPath());
            registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
            registration.setLoadOnStartup(webMvcProperties.getServlet().getLoadOnStartup());
            multipartConfig.ifAvailable(registration::setMultipartConfig);
            return registration;
        }
    }
}
```

如果配置了`/my`的`Servlet`，那么访问`/my`时，会调用自定义的`Servlet`（优先精确匹配）。

## 9.嵌入式servlet容器

内嵌服务器，就是手动调用启动服务器的代码。

### 9.1 切换嵌入式Servlet容器

默认支持的 webServer 有：`Tomcat`、`Jetty`、`Undertow`，`ServletWebServerApplicationContext` 容器启动会自动寻找 `ServletWebServerFactory` 并引导创建服务器。

切换容器时需要排除掉 Tomcat 的依赖：

```xml
<dependency>
	<groupId>org.springframework.boot</grouId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
    	<exclusion>
        	<groupId>org.springframework.boot</grouId>
   			<artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

**原理**

- `ServletWebServerFactoryAutoConfiguration` 配置类向容器中导入了 `ServletWebServerFactoryConfiguration`

  ```java
  @Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
  		ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
  		ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
  		ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
  public class ServletWebServerFactoryAutoConfiguration {
  ```

- `ServletWebServerFactoryConfiguration` 导入对应的服务器工厂

  导入各个容器都有条件，而默认导入了 tomcat 的包，所以只有 Tomcat 容器是满足条件的。

  ```java
  @Configuration(proxyBeanMethods = false)
  class ServletWebServerFactoryConfiguration {
  
      @Configuration(proxyBeanMethods = false)
      @ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
      @ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
      static class EmbeddedTomcat {
          @Bean
          TomcatServletWebServerFactory tomcatServletWebServerFactory(){}
      }
  
      @Configuration(proxyBeanMethods = false)
      @ConditionalOnClass({ Servlet.class, Server.class, Loader.class, WebAppContext.class })
      @ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
      static class EmbeddedJetty {
          @Bean
          JettyServletWebServerFactory JettyServletWebServerFactory(){}   
      }
  
      @Configuration(proxyBeanMethods = false)
      @ConditionalOnClass({ Servlet.class, Undertow.class, SslClientAuthMode.class })
      @ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
      static class EmbeddedUndertow {
          @Bean
          UndertowServletWebServerFactory undertowServletWebServerFactory(){}
      }
  }
  ```

- `SpringApplication` 执行 `run()` 时发现当前是 web 应用，会创建一个 web 版的 IOC 容器 `ServletWebServerApplicationContext`

- `SpringApplication` 启动的时候会创建程序上下文 `AnnotationConfigServletWebServerApplicationContext`，该类继承了 `ServletWebServerApplicationContext`

- 然后执行 `refreshContext()`，会最终执行程序启动的 `12` 大步骤

- 在执行 `onFresh()` 方法时会执行会调用 `createWebServer()` 方法

  ```java
  public class ServletWebServerApplicationContext {
      
      protected void onRefresh() {
          super.onRefresh();
          try {
              this.createWebServer();
          } catch (Throwable var2) {
              throw new ApplicationContextException("Unable to start web server", var2);
          }
      }
      
      private void createWebServer() {
  		WebServer webServer = this.webServer;
  		ServletContext servletContext = getServletContext();
  		if (webServer == null && servletContext == null) {
  			StartupStep createWebServer = this.getApplicationStartup().start("spring.boot.webserver.create");
              // 获得 ServletWebServerFactory
              // JettyServletWebServerFactory、TomcatServletWebServerFactory、UndertowServletWebServerFactory
  			ServletWebServerFactory factory = getWebServerFactory();
  			createWebServer.tag("factory", factory.getClass().toString());
              // 调用服务器工厂的 getWebServer() 方法创建 webServer
  			this.webServer = factory.getWebServer(getSelfInitializer());
  			createWebServer.end();
  			getBeanFactory().registerSingleton("webServerGracefulShutdown",
  					new WebServerGracefulShutdownLifecycle(this.webServer));
  			getBeanFactory().registerSingleton("webServerStartStop",
  					new WebServerStartStopLifecycle(this, this.webServer));
  		}
  		initPropertySources();
  	}
      
  }
  ```

### 9.2 定制Servlet容器

有以下三种方式：

- 修改配置文件`server.xxx.xxx`，例如 `server.tomcat.xxx`

- 实现 `WebServerFactoryCustomizer<ConfigurableServletWebServerFactory>`，自定义 `ConfigurableServletWebServerFactory`

  ```java
  @Component
  public class CustomizationBean implements WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {
      @Override
      public void customize(ConfigurableServletWebServerFactory server) {
          server.setPort(9000);
      }
  }
  ```
  
- 完全自定义时可以实现 `ServletWebServerFactory` 来自定义嵌入服务器

## 10.定制化原理

### 10.1 定制化的常见方式

#### 1.修改配置文件

#### 2.使用XXXXCustomizer

#### 3.编写自定义的配置类

`@Configuration` + `@Bean` 替换或者增加组件。

#### 4.实现WebMvcConfigurer

定义扩展 SpringMVC 底层功能。

| 提供方法                             | 核心参数                                | 功能                                                         | 默认                                                         |
| ------------------------------------ | --------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `addFormatters`                      | FormatterRegistry                       | **格式化器**：支持属性上 `@NumberFormat` 和 `@DatetimeFormat` 的数据类型转换 | GenericConversionService                                     |
| `getValidator`                       | 无                                      | **数据校验**：校验 Controller 上使用 `@Valid` 标注的参数合法性。需要导入 starter-validator 依赖 | 无                                                           |
| `addInterceptors`                    | InterceptorRegistry                     | **拦截器**：拦截收到的所有请求                               | 无                                                           |
| `configureContentNegotiation`        | ContentNegotiationConfigurer            | **内容协商**：支持多种数据格式返回。需要配合支持这种类型的 `HttpMessageConverter` | 支持 json                                                    |
| `configureMessageConverters`         | `List<HttpMessageConverter<?>>`         | **消息转换器**：标注 `@ResponseBody` 的返回值会利用 `MessageConverter` 直接写出去 | 8 个，支持 byte，string,multipart,resource，json             |
| `addViewControllers`                 | ViewControllerRegistry                  | **视图映射**：直接将请求路径与物理视图映射。用于无 java 业务逻辑的直接视图页渲染 | 无 `<mvc:view-controller>`                                   |
| `configureViewResolvers`             | ViewResolverRegistry                    | **视图解析器**：逻辑视图转为物理视图                         | ViewResolverComposite                                        |
| `addResourceHandlers`                | ResourceHandlerRegistry                 | **静态资源处理**：静态资源路径映射、缓存控制                 | ResourceHandlerRegistry                                      |
| `configureDefaultServletHandling`    | DefaultServletHandlerConfigurer         | **默认 Servlet**：可以覆盖 Tomcat 的 DefaultServlet。让 DispatcherServlet 拦截 `/` | 无                                                           |
| `configurePathMatch`                 | PathMatchConfigurer                     | **路径匹配**：自定义 URL 路径匹配。可以自动为所有路径加上指定前缀，比如 `/api` | 无                                                           |
| `configureAsyncSupport`              | AsyncSupportConfigurer                  | **异步支持**                                                 | TaskExecutionAutoConfiguration                               |
| `addCorsMappings`                    | CorsRegistry                            | **跨域**                                                     | 无                                                           |
| `addArgumentResolvers`               | `List<HandlerMethodArgumentResolver>`   | **参数解析器**                                               | mvc 默认提供                                                 |
| `addReturnValueHandlers`             | `List<HandlerMethodReturnValueHandler>` | **返回值解析器**                                             | mvc 默认提供                                                 |
| `configureHandlerExceptionResolvers` | `List<HandlerExceptionResolver>`        | **异常处理器**                                               | 默认 3 个 ExceptionHandlerExceptionResolver ResponseStatusExceptionResolver DefaultHandlerExceptionResolver |
| `getMessageCodesResolver`            | 无                                      | **消息码解析器**：国际化使用                                 | 无                                                           |

**原理**

- `WebMvcAutoConfiguration` 向容器中注入了一个 `EnableWebMvcConfiguration`，该类继承 `DelegatingWebMvcConfiguration`
- `DelegatingWebMvcConfiguration` 会中将容器中所有的 `WebMvcConfigurer` 注入进来

```java
public class WebMvcAutoConfiguration {
    @Configuration(proxyBeanMethods = false)
    @EnableConfigurationProperties({WebProperties.class})
    public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration implements ResourceLoaderAware {
        ...
    }
    ...
}

@Configuration(
    proxyBeanMethods = false
)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
    @Autowired(required = false)
    public void setConfigurers(List<WebMvcConfigurer> configurers) {
        if (!CollectionUtils.isEmpty(configurers)) {
            this.configurers.addWebMvcConfigurers(configurers);
        }
    }
    ...
}
```

#### 5.自定义核心组件

如果需要保持默认配置，但需要自定义核心组件，例如 `RequestMappingHandlerMapping`、`ExceptionHandlerExceptionResolver`，则可以注册一个 `WebMvcRegistrations` 组件

#### 6.全面接管配置

`@EnableWebMvc` + `WebMvcConfigurer` 全面接管 SpringMVC，所有规则都需要重新自己配置

**原理**

`@EnableWebMvc` 会引入 `DelegatingWebMvcConfiguration`

```java
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```

`DelegatingWebMvcConfiguration` 只保证 SpringMVC 最基本的应用：

- 会加载所有的 `WebMvcConfigurer`，所有功能都由 `WebMvcConfigurer` 配置

- 其父类 `WebMvcConfigurationSupport` 会向容器中导入基本组件，例如 `RequestMappingHandlerMapping`、`mvcContentNegotiationManager` 等

而 `WebMvcAutoConfiguration` 只会在容器中没有 `WebMvcConfigurationSupport` 时才会被加载：

```java
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
public class WebMvcAutoConfiguration {
```

所以，`@EnableWebMvc` 会导致默认的 `WebMvcAutoConfiguration` 失效。

### 10.2 WebMvcAutoConfiguration原理

```java
@AutoConfiguration(after = {DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class, ValidationAutoConfiguration.class})
// 如果是 Web 应用则生效
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class})
@ConditionalOnMissingBean({WebMvcConfigurationSupport.class})
@AutoConfigureOrder(-2147483638)
@ImportRuntimeHints({WebResourcesRuntimeHints.class})
public class WebMvcAutoConfiguration {
    
    /**
     * 支持 Restful 的 filter
     */
    @Bean
    ...
    public OrderedHiddenHttpMethodFilter hiddenHttpMethodFilter() {
        return new OrderedHiddenHttpMethodFilter();
    }

    /**
     * 支持表单内容 Filter，GET、POST 请求可以携带数据，PUT 和 DELETE 请求体数据会被忽略
     */
    @Bean
    ...
    public OrderedFormContentFilter formContentFilter() {
        return new OrderedFormContentFilter();
    }
    
    /**
     * 新特性，用于配置错误详情处理器，处理 SpringMVC 内部场景异常
     */
    @Configuration(proxyBeanMethods = false)
    static class ProblemDetailsErrorHandlingConfiguration {
    	...
    }
    
    
    /**
     * 注册一个 WebConfigurer 定义默认行为，还添加了一些 Bean
     */
    @Configuration(
        proxyBeanMethods = false
    )
    @Import({WebMvcAutoConfiguration.EnableWebMvcConfiguration.class})
    // WebMvcProperties：prefix = "spring.mvc"
    // WebProperties：prefix = "spring.web"
    @EnableConfigurationProperties({WebMvcProperties.class, WebProperties.class})
    public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer, ServletContextAware {
        
    	...
        
        // 添加一个视图解析器：InternalResourceViewResolver
        @Bean
        public InternalResourceViewResolver defaultViewResolver() {
            ...
        }

        // 添加一个视图解析器：BeanNameViewResolver
        @Bean
        public BeanNameViewResolver beanNameViewResolver() {
            ...
        }
        
        // 添加一个内容协商视图解析器
        @Bean
        public ContentNegotiatingViewResolver viewResolver(BeanFactory beanFactory) {
            ...
        }
        
        // 添加一个请求上下文过滤器，可以在任意位置使用 (ServletRequestAttributes) RequestContextHolder.getRequestAttributes() 获取当前请求和响应信息
        @Bean
        public static RequestContextFilter requestContextFilter() {
            return new OrderedRequestContextFilter();
        }
        
        // 定义了一系列静态资源规则
    }
    
    /**
     * 注册一个 EnableWebMvcConfiguration
     */
    @Configuration(
        proxyBeanMethods = false
    )
    @EnableConfigurationProperties({WebProperties.class})
    public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration implements ResourceLoaderAware {
        
        /**
         * 注册一个 RequestMappingHandlerAdapter
         */
        protected RequestMappingHandlerAdapter createRequestMappingHandlerAdapter() {
            ...
        }
        
        /**
         * 注册 ExceptionHandlerExceptionResolver
         */
        protected ExceptionHandlerExceptionResolver createExceptionHandlerExceptionResolver() {
            ...
        }
        
        /**
         * 提供数据校验功能
         */
        @Bean
        public Validator mvcValidator() {
            return !ClassUtils.isPresent("jakarta.validation.Validator", this.getClass().getClassLoader()) ? super.mvcValidator() : ValidatorAdapter.get(this.getApplicationContext(), this.getValidator());
        }
        
        /**
         * 注册内容协商管理器
         */
        @Bean
        public ContentNegotiationManager mvcContentNegotiationManager() {
            ...
        }
        
        /**
         * 注册一个欢迎页处理器映射
         */
        @Bean
        public WelcomePageHandlerMapping welcomePageHandlerMapping(ApplicationContext applicationContext, FormattingConversionService mvcConversionService, ResourceUrlProvider mvcResourceUrlProvider) {
            ...
        }
        
        /**
         * 注册一个国际化解析器
         */
        @Bean
        @ConditionalOnMissingBean(name = {"localeResolver"})
        public LocaleResolver localeResolver() {
            ...
        }
        
        /**
         * 注册一个服务用于处理日期转换
         */
        @Bean
        public FormattingConversionService mvcConversionService() {
            ...
        }
        
        ...
    }
    
    ...
}
```

## 11.路径匹配

Spring5.3 之后加入了更多的请求路径匹配的实现策略：以前只支持 `AntPathMatcher` 策略, 现在提供了 `PathPatternParser` 策略，并且可以让我们指定到底使用那种策略。

### 11.1 Ant风格路径用法

Ant 风格的路径模式语法具有以下规则：

- `*`：表示**任意数量**的字符
- `?`：表示任意**一个字符**
- `**`：表示**任意数量的目录**
- `{}`：表示一个命名的模式**占位符**
- `[]`：表示**字符集合**，例如 [a-z] 表示小写字母

例如：

- `*.html` 匹配任意名称，扩展名为 .html 的文件

- `/folder1/*/*.java` 匹配在 folder1 目录下的任意两级目录下的 .java 文件

- `/folder2/**/*.jsp` 匹配在 folder2 目录下任意目录深度的 .jsp 文件

- `/{type}/{id}.html` 匹配任意文件名为 {id}.html，在任意命名的 {type} 目录下的文件

- `/a*/b?/{p1:[a-f]+}`，匹配 a 开头的字符 + b 和其后任意一个字符 + a 到 f 的任意 0 至多个字符（占位符名称为 p1）

  ```java
  @GetMapping("/a*/b?/{p1:[a-f]+}")
  public String hello(HttpServletRequest request, @PathVariable("p1") String path) {
      log.info("路径变量p1： {}", path);
      //获取请求路径
      String uri = request.getRequestURI();
      return uri;
  }
  ```

注意：Ant 风格的路径模式语法中的特殊字符需要转义，如：

- 要匹配文件路径中的星号，则需要转义为 `\\*`
- 要匹配文件路径中的问号，则需要转义为 `\\?`

### 11.2 模式切换

- PathPatternParser 在 jmh 基准测试下，有 6~8 倍吞吐量提升，降低 30%~40% 空间分配率
- PathPatternParser 兼容 AntPathMatcher 语法，并支持更多类型的路径模式
- PathPatternParser 的  `**` 多段匹配的支持**仅允许在模式末尾使用**

总结： 

- 默认的路径匹配规则是 PathPatternParser

- 如果路径中间需要有 `**`，需要替换成 ant 风格路径

  ```properties
  # 改变路径匹配策略：
  # ant_path_matcher 老版策略
  # path_pattern_parser 新版策略
  spring.mvc.pathmatch.matching-strategy=ant_path_matcher
  ```

## 12.国际化

国际化的自动配置参照 `MessageSourceAutoConfiguration`。

- **配置**（可选）

  默认需要将国际化文件放在 `resources` 目录下，可以手动更改。

  配置类

  ```java
  @Configuration
  public class MessageI18NConfig {
      @Bean
      public ResourceBundleMessageSource resourceBundleMessageSource() {
          ResourceBundleMessageSource resourceBundleMessageSource = new ResourceBundleMessageSource();
          // 从 resources/messages 下查找以 messages 开头的文件
          resourceBundleMessageSource.setBasename("messages/messages");
          resourceBundleMessageSource.setDefaultLocale(Locale.CHINA);
          resourceBundleMessageSource.setDefaultEncoding("UTF-8");
          return resourceBundleMessageSource;
      }
  }
  ```

  配置文件

  ```properties
  spring.messages.basename=messages.messages
  spring.messages.encoding=UTF-8
  ```

- **添加 properties 文件**

  ```properties
  # messages_en_US.properties
  E001={0} Hello!
  
  # messages_zh_CN.properties
  E001={0} 你好！
  ```

- **代码中获取值**

  ```java
  @Autowired
  ResourceBundleMessageSource resourceBundleMessageSource;
  
  @GetMapping("/inter")
  public String inter(HttpServletRequest request) {
      return resourceBundleMessageSource.getMessage("E001", new String[]{"zhang3"}, request.getLocale());
  }
  ```


## 13.新特性

### 1.ProblemDetails

[RFC 7807](https://www.rfc-editor.org/rfc/rfc7807)

默认响应错误的 json：

```json
{
    "timestamp": "2023-04-18T11:13:05.515+00:00",
    "status": 405,
    "error": "Method Not Allowed",
    "trace": "org.springframework.web.HttpRequestMethodNotSupportedException: Request method 'POST' is not supported\r\n\tat...",
    "message": "Method 'POST' is not supported.",
    "path": "/list"
}
```

开启 ProblemDetails 后返回，使用新的 MediaType：`application/problem+json`

```json
{
    "type": "about:blank",
    "title": "Method Not Allowed",
    "status": 405,
    "detail": "Method 'POST' is not supported.",
    "instance": "/list"
}
```

**原理**

`WebMvcAutoConfiguration` 注册了一个 `ProblemDetailsErrorHandlingConfiguration` 组件。

```java
@Configuration(proxyBeanMethods = false)
// 需要配置一个属性 spring.mvc.problemdetails.enabled=true
@ConditionalOnProperty(prefix = "spring.mvc.problemdetails", name = "enabled", havingValue = "true")
static class ProblemDetailsErrorHandlingConfiguration {

    @Bean
    @ConditionalOnMissingBean(ResponseEntityExceptionHandler.class)
    ProblemDetailsExceptionHandler problemDetailsExceptionHandler() {
        return new ProblemDetailsExceptionHandler();
    }

}
```

1. `ProblemDetailsExceptionHandler` 是一个 `@ControllerAdvice` 用于集中处理系统异常

2. 如果系统出现以下异常，会被 SpringBoot 支持以 `RFC 7807` 规范方式返回错误数据

   ```java
   @ExceptionHandler({
       HttpRequestMethodNotSupportedException.class, //请求方式不支持
       HttpMediaTypeNotSupportedException.class,
       HttpMediaTypeNotAcceptableException.class,
       MissingPathVariableException.class,
       MissingServletRequestParameterException.class,
       MissingServletRequestPartException.class,
       ServletRequestBindingException.class,
       MethodArgumentNotValidException.class,
       NoHandlerFoundException.class,
       AsyncRequestTimeoutException.class,
       ErrorResponseException.class,
       ConversionNotSupportedException.class,
       TypeMismatchException.class,
       HttpMessageNotReadableException.class,
       HttpMessageNotWritableException.class,
       BindException.class
           })
   ```

### 2.函数式Web

SpringMVC 5.2 以后 允许我们使用**函数式**的方式定义 Web 的请求处理流程。

Web请求处理的方式：

1. `@Controller + @RequestMapping`：**耦合式** （**路由**、**业务**耦合）
2. **函数式Web**：分离式（路由、业务分离）

核心类

- `RouterFunction`：定义路由信息，发什么请求，谁来处理
- `RequestPredicate`：请求方式（GET、POST）、请求参数
- `ServerRequest`：封装请求数据
- `ServerResponse`：封装响应数据

配置类

```java
@Configuration(proxyBeanMethods = false)
public class MyRoutingConfiguration {

    // 定义请求规则
    private static final RequestPredicate ACCEPT_JSON = RequestPredicates.accept(MediaType.APPLICATION_JSON);

    @Bean
    public RouterFunction<ServerResponse> routerFunction(MyUserHandler userHandler) {
        return RouterFunctions.route()
                .GET("/{user}", ACCEPT_JSON, userHandler::getUser)
                .GET("/{user}/customers", ACCEPT_JSON, userHandler::getUserCustomers)
                .DELETE("/user", ACCEPT_JSON, userHandler::deleteUser)
                .build();
    }

}
```

处理器类

```java
@Component
public class MyUserHandler {

    public ServerResponse getUser(ServerRequest request) {
        // 业务处理
        // 获取路径参数
        String user = request.pathVariable("user");
        ...
        return ServerResponse.ok().body(xxx);
    }

    public ServerResponse getUserCustomers(ServerRequest request) {
        // 业务处理
        ...
        return ServerResponse.ok().body(xxx);
    }

    public ServerResponse deleteUser(ServerRequest request) {
        // 业务处理
        // 获取请求体
        Person body = request.body(Person.class);
        ...
        return ServerResponse.ok().build();
    }

}
```

`.body()` 跟之前的 `@ResponseBody` 原理相同。

