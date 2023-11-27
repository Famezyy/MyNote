# 第5章_Web开发

## 1.静态资源访问

**静态资源目录**

静态资源放在类路径下：`classpath:/static`、`classpath:/public`、`classpath:/resources`、`classpath:/META-INF/resources`。

访问时需要访问：当前项目根路径 + / + 静态资源名。

> **原理**
>
> 收到请求时，先去找对应的`Controller`看谁能处理。找不到的所有请求都交给静态资源处理器。静态资源也找不到时则相应 404 页面。

**改变默认的静态资源路径**

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

**自定义Favicon（浏览器图标）**

将`Favicon.ico`放在静态资源目录下即可。

**修改启动图**

```yaml
spring:
  banner:
    image:
      location: classpath:2.png
```

**静态资源配置原理**

`WebMvcAutoConfiguration`有一个内部静态类`WebMvcAutoConfigurationAdapter`，实现了`WebMvcConfigurer`。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
                     ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {
    
    @Import(EnableWebMvcConfiguration.class)
    // WebMvcProperties：prefix = "spring.mvc"
    // WebProperties：prefix = "spring.web"
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
			addResourceHandler(registry, this.mvcProperties.getStaticPathPattern(), (registration) -> {
                /**
                 * CLASSPATH_RESOURCE_LOCATIONS = { "classpath:/META-INF/resources/", "classpath:/resources/", "classpath:/static/", "classpath:/public/" };
                 */
				registration.addResourceLocations(this.resourceProperties.getStaticLocations());
				if (this.servletContext != null) {
					ServletContextResource resource = new ServletContextResource(this.servletContext, SERVLET_LOCATION);
					registration.addResourceLocations(resource);
				}
			});
		}
        
    }
    
}
```

**欢迎页的处理规则**

```java
final class WelcomePageHandlerMapping extends AbstractUrlHandlerMapping {
	WelcomePageHandlerMapping(TemplateAvailabilityProviders templateAvailabilityProviders,
			ApplicationContext applicationContext, Resource welcomePage, String staticPathPattern) {
        // 要使用欢迎页功能，必须是 /**
		if (welcomePage != null && "/**".equals(staticPathPattern)) {
			logger.info("Adding welcome page: " + welcomePage);
            // 底层使用转发机制
			setRootViewName("forward:index.html");
		}
		else if (welcomeTemplateExists(templateAvailabilityProviders, applicationContext)) {
			logger.info("Adding welcome page template: index");
			setRootViewName("index");
		}
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

- 标注`@ModelAttribute`注解且为对象类型

- 标注`@ResponseBody`注解

#### 2.默认的MessageConverter支持类型

- ByteArrayHttpMessageConverter：Byte
- StringHttpMessageConverter：String（UTF-8）
- StringHttpMessageConverter：String（ISO-8859-1）
- ResourceRegionHttpMessageConverter：ResourceRegion
- SourceHttpMessageConverter：Resource
- AllEncompassingFormHttpMessageConverter：MultiValueMap
- MappingJackson2HttpMessageConverter：所有类型
- MappingJackson2HttpMessageConverter：所有类型
- Jaxb2RootElementHttpMessageConverter：注解方式 xml 处理的

#### 3.响应JSON

调用返回值处理器的`supportsReturnType()`方法查找支持的返回值处理器。

当标注了`@ResponseBody`时，由`RequestResponseBodyMethodProcessor`来解析返回值。

然后遍历所有的`MessageConverter`的`canWrite()`方法，最终由`AbstractJackson2HttpMessageConverter`来负责写出操作。

底层利用 jackon 的 objectMapper 转换。

### 3.2 内容协商

获得浏览器传入的接受的媒体类型，返回对应的媒体类型。

#### 1.开启支持xml内容协商

**引入xml依赖**

```xml
<dependency>
  <groupId>com.fasterxml.jackson.dataformat</groupId>
  <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

#### 2.开启浏览器参数方式内容协商

浏览器默认使用基于请求头的策略`HeaderContentNegotiationStrategy`，获得`headers`中的`ACCEPT`参数。

添加参数方式策略`ParameterContentNegotiationStrategy`，但该策略仅支持`xml`和`json`。

```yaml
spring:
	mvc:
    	contentnegotiation:
      		favor-parameter: true  #会添加基于参数的内容协商策略
```

此时发送请求：`http://localhost:8080/test/person?format=json`或者`http://localhost:8080/test/person?format=xml`即可。

### 3.3 原理

- 执行`handler()`方法处理返回值时，会先获得浏览器传入的接受的媒体类型。默认使用基于请求头的内容协商策略，会获得`headers`中的`ACCEPT`参数
- 然后获取可产生的媒体类型，遍历所有`messageConverter`，调用相应的`converter.canWrite()`方法查询支持写入相应的类型的`converter`
- 匹配接受的媒体类型和产生的媒体类型，选择最合适的`mediaType`
- 循环遍历所有`messageConverters`查找支持相应类型转换的`Converters`
- 调用对应转换器的`write()`方法完成写入

### 3.4 自定义MessageConverter

方法1：直接向容器中注册一个`HttpMessageConverter`类型的组件

```java
@Component
public class MyConverter implements HttpMessageConverter<Bservice> {
    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        return false;
    }

    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        return clazz.isAssignableFrom(Bservice.class);
    }

    // 声明支持的 MediaType
    @Override
    public List<MediaType> getSupportedMediaTypes() {
        return MediaType.parseMediaTypes("application/youyi");
    }


    @Override
    public Bservice read(Class<? extends Bservice> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        return null;
    }

    @Override
    public void write(Bservice bservice, MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        String data = bservice.getA() + bservice.getB();
        OutputStream body = outputMessage.getBody();
        body.write(data.getBytes());
    }
}
```

方法2：不使用`@Component`注解，在`WebMvcConfigurer`添加

```java
@Configuration
public class MyMapperConfiguration implements WebMvcConfigurer {

    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new MyConverter());
    }
}
```

**设置支持基于参数的内容协商**

需要使自定义的`MessageConverter`支持基于参数的内容协商时，需要配置内容协商策略。因为`ParameterContentNegotiationStrategy`策略仅支持`xml`和`json`。

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

但此时会覆盖原有的内容协商管理器，使基于请求头的策略也失效。此时若 mediaType 无法匹配，会默认为全匹配`*/*`，此时会与`json`的格式匹配上。

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

如果不是一个**RestController**则返回 String 时会自动将其视为视图并进行解析，如果无法解析则抛出相关错误。

### 4.1 视图解析

`SpringBoot`默认不支持 JSP，需要引入第三方模块引擎技术实现页面渲染。

**原理流程**

- 在处理派发结果时，会调用`DispatcherServlet`的`render()`方法进行页面渲染（出错或者返回值为 JSON 时，mv 为 null，不会调用该方法）

- 根据 view 名称创建 view 对象，一般由`ContentNegotiationViewResolver`解析

  - error：`ErrorMvcAutoconfiguration$StaticView`

  - `forward:/**`：`InternalResourceView`

    > `request.getRequestDispatcher(path).forward(request, response)`

  - `redirect:/**`：`RedirectView`

    > `response.sendRedirect(encodedURL)`

  - string：默认没有解析器，可添加 thymeleaf 作为解析器

- 调用对应 view 的`render()`方法进行渲染

### 4.2 模板解析-thymeleaf

Thymeleaf is a modern server-side Java template engine for both web and standalone environments, capable of processing HTML, XML, JavaScript, CSS and even plain text.

#### 1.基本语法

##### 1.1 表达式

| 表达式名字 | 语法                                                    | 用途                           |
| ---------- | ------------------------------------------------------- | ------------------------------ |
| 变量取值   | ${...}                                                  | 获取请求域、session域、对象    |
| 选择变量   | *{...}                                                  | 获取上下文对象值               |
| 消息       | #{...}                                                  | 获取国际化等值                 |
| 连接       | @{...}：自动加上当前路径<br />@{/...}：自动加上访问路径 | 生成链接                       |
| 片段表达式 | ~{...}                                                  | jsp.include 作用，引入公共页面 |

##### 1.2 字面量

文本值：'text'

数字：1

布尔值：true，false

控制：null

变量：x，y（变量名不能有空格）

##### 1.3 文本操作

字符串拼接：+

##### 1.4 数字运算

+、-、*、/、%

##### 1.5 布尔运算

运算符：and、or

##### 1.6 比较运算

\>、<、>=、<=、==、!=

##### 1.7 条件运算

- (if) ? (then)
- (if) ? (then) : (else)
- (value) ? (defaultvalue)

##### 1.8无标签时的行内使用

```html
<img src="1.png" />
[[${session.loginUser.username}]]
<span class="create"></span>
```

#### 2.设置属性值-th:attr

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

替代写法：th:**

```html
<form action="subscribe.html" th:action="@{/subscribe}">
<input type="submit" value="Subscribe!" th:value="#{subscribe.submit}" />
```

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

自动配置好的策略

- 所有 thymeleaf 的配置值都在 ThymeleafProperties
- 配置好了 SpringTemplateEngine
- 配置好了 ThymeleafViewResolver

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

内嵌服务器，就是手动把启动服务器的代码调用

### 9.1 切换嵌入式Servlet容器

默认支持的 webServer 有：`Tomcat`、`Jetty`、`Undertow`，`ServletWebServerApplicationContext`容器启动寻找`ServletWebServerFactory`并引导创建服务器。

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

- `ServletWebServerFactoryAutoConfiguration`配置类向容器中导入了`ServletWebServerFactoryConfiguration`

  ```java
  @Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
  		ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
  		ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
  		ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
  public class ServletWebServerFactoryAutoConfiguration {
  ```

- `ServletWebServerFactoryConfiguration`导入对应的服务器（默认导入 tomcat 包）

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

- SpringApplication 执行 run() 时发现当前是 web 应用，会创建一个 web 版的 IOC 容器`ServletWebServerApplicationContext`

- `ServletWebServerApplicationContext`启动的时候寻找`ServletWebServerFactory`，并创建相应的 webServer

  ```java
  public class ServletWebServerApplicationContext {
      private void createWebServer() {
  		WebServer webServer = this.webServer;
  		ServletContext servletContext = getServletContext();
  		if (webServer == null && servletContext == null) {
  			StartupStep createWebServer = this.getApplicationStartup().start("spring.boot.webserver.create");
              // 获得 ServletWebServerFactory
              // JettyServletWebServerFactory、TomcatServletWebServerFactory、UndertowServletWebServerFactory
  			ServletWebServerFactory factory = getWebServerFactory();
  			createWebServer.tag("factory", factory.getClass().toString());
              // 获得 webServer
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

有以下两种方式：

- 修改配置文件`server.xxx`

- 实现`WebServerFactoryCustomizer<ConfigurableServletWebServerFactory>`，自定义`ConfigurableServletWebServerFactory`

  ```java
  @Component
  public class CustomizationBean implements WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {
      @Override
      public void customize(ConfigurableServletWebServerFactory server) {
          server.setPort(9000);
      }
  }
  ```

## 10.定制化原理

### 10.1 定制化的常见方式

1. 修改配置文件

2. XXXXCustomizer

3. 编写自定义的配置类`@Configuration` + `@Bean`替换或者增加组件

   - 可以实现`WebMvcConfigurer`可定制化大部分 web 功能
     - 开启矩阵变量
     - 添加`converter`
     - 添加内容协商策略
     - 添加`MessageConverter`
     - 配置拦截器

4. `@EnableWebMvc` + `@WebMvcConfigurer` + `@Bean` 可以全面接管 SpringMVC，所有规则全部重新配置

   **原理**

   - `@EnableWebMvc`会引入`DelegatingWebMvcConfiguration`

     ```java
     @Import(DelegatingWebMvcConfiguration.class)
     public @interface EnableWebMvc {
     }
     ```

   - `DelegatingWebMvcConfiguration`只保证 SpringMVC 最基本的应用

     - 会加载所有的`WebMvcConfigurer`，所有功能都由`WebMvcConfigurer`配置
     - 其父类`WebMvcConfigurationSupport`会向容器中导入基本组件，例如`RequestMappingHandlerMapping`、`mvcContentNegotiationManager`等

   - 而`WebMvcAutoConfiguration`只会在容器中没有`WebMvcConfigurationSupport`时才会被加载

     ```java
     @ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
     @AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
     @AutoConfigureAfter({ DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
     		ValidationAutoConfiguration.class })
     public class WebMvcAutoConfiguration {
     ```

   所以，`@EnableWebMvc`会导致默认的`WebMvcAutoConfiguration`失效。

