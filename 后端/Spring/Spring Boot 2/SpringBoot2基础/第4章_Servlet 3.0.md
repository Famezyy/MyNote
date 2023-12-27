# 第4章_Servlet 3.0

`Servlet` 是一套标准，`Servlet` 容器产品按照这套标准提供一个运行时环境。这个运行时环境支持将 Http 请求以 Java 对象的方式传递给开发人员编写的业务类，并且支持开发人员以 Java 对象的方式返回数据。运行时环境会将返回数据封装为 Http 响应报文发送给客户端。

`Servlet` 在客户端 HTTP 请求与开发人员编写的 `Servlet` 具体业务类之间定义了一套标准规范，而容器厂家根据标准规范提供具体实现。这样开发人员只需关注具体的业务逻辑本身即可，不必再关心网络传输、协议相关的处理。

## 1.简介

### 1.1 容器

"容器"在软件领域可以理解为**装载**、**隔离**和**实现**，例如 Docker、Spring IOC 容器、Tomcat 等都是容器，用来统一管理资源。

- **装载**

  容器能够装载 Servlet 相关的技术组件，为 Servlet 技术组件提供了运行时环境。单独基于 Servlet 组件开发的应用程序是**无法独立运行**的，必须通过容器装载 Servlet 相关的其他组件，才能对外提供网络服务能力。

  容器装载 Servlet 技术组件的过程成为"部署"。容器与 Servlet 技术组件互相适配的根基就是《Java Servlet 标准规范》。开发人员按照标准规范的要求编写代码，组织目录结构，编写配置文件，最后打包成文件。

  容器产品按照标准规范解析打包文件，根据配置文件加载各个组件，实例并初始化它们，按照标准中规定的生命周期管理这些对象，在事件发生时触发对应的监听器。

  容器启动后监听服务器的一个端口，接收到网络请求，解析成符合 Servlet 标准的请求对象，派发给特定的 Servlet 实例执行业务逻辑，最后将结果通过端口发送给客户端，完成网络请求-响应过程。

- **隔离**

  容器相当于一层中间件，屏蔽了外部复杂环境的差异性。Java Web 应用部署在 Servlet 容器内，Servlet 容器产品安装在服务器的操作系统中。这样，开发人员就不必关心服务器环境的差别，只需要关注封装后的请求 `Request` 对象和响应 `Response` 对象的处理。

  同时，基于标准规范实现容器产品，可以使得应用程序无缝迁移。

  容器还可以帮助我们解决与业务无关的通用技术，如协议解析、对象转换、对象管理、生命周期管理等，使开发人员可以专注于业务逻辑的编写。

- **实现**

  Servlet 并不是一个技术组件而是一套标准，包含了多个组件和技术要求，Servlet 容器厂商需要实现这些要求。我们在开发 JavaWeb 应用的时候，通常会引入 `javax.servlet-api.jar`，或者在 SpringBoot 中导入 `spring-boot-start-web.jar`，其中会引入 `tomcat-embed-core.jar`，这两个包里都会包含 Servlet 的标准库。

  查看 JAR 包就会发现绝大部分是接口，而各个容器软件就负责实现这些接口。

### 1.2 Servlet核心类

<img src="img/第4章_Servlet 3.0/image-20231227163549774.png" alt="image-20231227163549774" style="zoom:67%;" />

实际开发中可以继承 `HttpServlet` 来实现自定义的 `Servlet`。`Servlet` 接口中 `service()` 方法用于执行实际的业务逻辑，而 `HttpServlet` 中将该方法根据请求方法细分成了不同的方法来处理，例如 `doGet()`、`doPost()` ，开发时可以重写 `doGet()` 和 `doPost()` 方法来实现业务逻辑。

编写完 `Servlet` 后需要注册到上下文中，同时要绑定 url。

### 1.3 请求流程

<img src="img/第4章_Servlet 3.0/image-20231227163710319.png" alt="image-20231227163710319" style="zoom: 67%;" />

- Servelet 容器产品（如 Tomcat），又称为 Java Web 中间件，负责监听服务端的端口，接收特定协议的网络请求
- 容器将每次请求都封装成一对 `HttpServletRequest` 和 `HttpServletResponse` 对象，同时绑定会话对象
- `Request` 对象和 `Response` 对象向后传递，根据 URL 匹配到一个 `Filter` 链，依次执行每个 `Filter` 的 `doFilter` 方法
- 再根据 URL 匹配到最多一个 `Servlet` 对象，将 `Request` 和 `Response` 交给该 `Servlet` 对象的 `service()` 方法进行处理
- `Servlet` 处理好后将结果放入 `Response` 对象中，再将 `Request` 和 `Response` 对象以倒叙方式传递给 `Filter` 链
- 最后 `Servlet` 容器将 `Response` 对象转换为 Http 响应报文，发送回客户端

在整个过程中，`Servlet` 容器会按照 Java Servlet 标准规范中定义的时机，生成各种事件，由注册的监听器进行处理。

> **注意**
>
> 在以前 `Servlet` 往往被用来实现业务功能，但是现在更多的是作为**统一入口**，负责解析请求，将不同的请求分发到不同的 `Controller`，由 `Controller` 负责执行具体业务逻辑。

## 2.Request与Response

`HttpServletRequest` 和 `HttpServletResponse` 是请求对象和响应对象的接口，分别继承了 `ServletRequest` 和 `ServletResponse`，其中集成了一些操作 HTTP 请求和响应的方法。实际开发中不需要开发人员去实现接口，Servlet 容器提供了默认实现，只需了解 API 的使用即可。

### 2.1 字符集

基于 HTTP 通信时，要注意客户端和服务端的字符集一致，最好统一成 `UTF-8`。在 `Servlet` 标准中，能够设置字符集的方法有以下 2 种：

- 在 `ServletCOntext` 中设置全局字符集
- `Request` 和 `Response` 对象中设置，只针对当前请求有效

同时可以通过 `ContentType` 告知对方自己发送的编码集。`ContentType` 也叫做 `MIME` 类型，是 HTTP 协议中的一个报文头。

常见的请求 `ContentType` 有以下几种：

- `application/x-www-form-urlencoded`：表单数据
- `application/json; charset=UTF-8`：JSON 数据
- `multipart/form-data`：上传文件

常见的响应 `ContentType` 有以下几种：

- `text/html; charset=UTF-8`：默认值，代表返回的是普通文本
- `application/json; charset=UTF-8`：JSON 数据
- `image/jpeg`：JPEG 格式图片

### 2.2 获取参数

在 `Request` 中调用 `getParameter` 系列方法可以获得 `Map<Stirng, List<String>>` 类型的请求参数集合，该集合中放入了 query parameter 和表单请求数据参数（`ContentType` 为 `application/x-www-form-urlencoded`）。

> **注意**
>
> 如果先调用了 `getInputStream()` 方法则会将输入流消费完，从而导致其他方法（如 `getParameter()`）失效。

## 3.Filter

<img src="img/第4章_Servlet 3.0/image-20231227225835521.png" alt="image-20231227225835521" style="zoom: 50%;" />

`Filter` 是**责任链模式**的典型应用，其功能更像是一个流水线而不是过滤器，它可以获取 `Request` 和 `Response` 并进行验证和操作，放行时需要执行 `chain.doFilter(req, resp)` 将其传递给下一个 `Filter`。`Filter` 常用来进行认证限权、日志记录、装饰 `Request`、`Response`、`Session` 等。

## 4.ServletContext

`ServletContext` 是在 JVM 上的 Web 应用的唯一上下文信息。

### 4.1 获取ServletContext

获取方式有以下三种：

- `ServletContainerInitializer`

  这是 `Servlet` 标准规范中定义的一个接口，该接口有一个方法：

  ```java
  public void onStartup(Set<Class<?>> c, ServletContext ctx)
  ```

  使用时需实现该接口并重写 `onStartup()` 方法，最后将其以 SPI 的方式注册。Servlet 容器启动后，会扫描当前应用中的每一个 jar 包的 `ServletContainerInitializer` 的实现，使用时有以下 2 个步骤：
  
  1. 实现 `ServletContainerInitializer`
  
     ```java
     // 容器启动时会将 HandlesTypes 指定类型（接口或类）的下面的子类，实现类，子接口等全部传递过来
     @HandlesTypes(Object.class)
     public class MyServletContainerInitializer implements ServletContainerInitializer {
         /**
          * 应用启动时，会运行 onStartup 方法
          * @param set：HandlesTypes 指定的类型的所有子类型
          * @param servletContext：代表当前 web 应用的 ServletContext，一个 web 应用对应一个 ServletContext
          * @throws ServletException
          */
         @Override
         public void onStartup(Set<Class<?>> set, ServletContext servletContext) throws ServletException {
             System.out.println("class：");
             for (Class<?> aClass : set) {
                 System.out.println(aClass.getName());
             }
         }
     }
     ```
  
  2. 创建文件：`META-INF/services/javax.servlet.ServletContainerInitializer`，文件的内容就是 `ServletContainerInitializer` 的实现类的全类名（Idea 要放在 `/resources` 路径下）
  
     ```bash
     com.example.servlet.MyServletContainerInitializer
     ```
  
  容器在启动应用时，会扫描当前应用每一个 JAR 包里面的 `META-INF/Services/javax.servlet.ServletContainerInitializer` 指定的实现类，启动并运行这个实现类方法。
  
- `ServletContextListener`

  这也是 Servlet 标准规范中定义的一个接口，开发人员可以实现该接口完成对 `ServletContext` 声名周期事件的监听。

  ```java
  @WebListener
  public class MyServletContextListener implements ServletContextListener {
      /**
       * 该方法在 ServletContext 完成初始化时执行
       */
      @Override
      public void contextInitialized(ServletContextEvent sce) {
  	ServletContextListener.super.contextInitialized(sce);
      }
  
      /**
       * 该方法在 ServletContext 销毁时执行
       */
      @Override
      public void contextDestroyed(ServletContextEvent sce) {
  	ServletContextListener.super.contextDestroyed(sce);
      }
  }
  ```
  
- `getServletContext()`

  `HttpServlet` 继承自 `GenericServlet`，在 `GenericService` 中提供了 `ServletContext getServletContext()` 方法可以直接获取 `ServletCOntext` 实例对象。此外，`Filter` 的 `init()` 方法参数为 `FilterConfig`，该接口也提供了 `getServletContext()` 方法。

### 4.2 注册Web组件

```java
// 注册 Servlet
ServletRegistration.Dynamic userServlet = servletContext.addServlet("userServlet", new UserServlet());
// 配置 Servlet 映射信息
userServlet.addMapping("/user");

// 注册 Listener
servletContext.addListener(UserListener.class);

// 注册 Filter
FilterRegistration.Dynamic userFilter = servletContext.addFilter("UserFilter", UserFilter.class);
// 配置 Filter 映射信息
userFilter.addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST), true, "/*");
```

## 5.与SpringMVC整合

1. Web 容器启动时，会扫描每个 jar 包下的 META-INFO/services/javax.servlet.ServletContainerInitializer

2. 加载这个文件指定的类：SpringServletContainerInitializer
3. Spring 的应用启动后会加载 WebApplicationInitializer 接口下的所有组件
4. 并且为 WebApplicationInitializer 这些组件创建对象（组件不是接口，不是抽象类）
   1. AbstractContextLoaderInitializer：创建根容器，ContextLoaderListener
   2. AbstractDispatcherServletInitializer
      1. 创建一个 web 的 IOC 容器：createServletApplicationContext()
      2. 创建一个 DispatcherServlet：createDispatcherServlet()
      3. 将创建的 DispatcherServlet 添加到 ServletContext 中
   3. AbstractAnnotationConfigDispatcherServletInitializer：注解方式配置的 DispatcherServlet 初始化器
      1. 创建根容器
      2. getServletConfigClasses()：传入一个配置类
      3. 创建一个 web 的 IOC 容器：createServletApplicationContext()
         - 获取配置类：getServletConfigClasses()

**总结**

1. 以注解方式启动 SpringMVC，继承 AbstractAnnotationConfigDispatcherServletInitializer

2. 实现抽象方法指定 DispatcherServlet 的配置信息

```java
// web 容器启动时候创建对象，调用方法来初始化容器以及前端控制器
public class MyWebInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    // 获取根容器的配置类，父容器（Spring 的配置文件）
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{RootConfig.class};
    }

    // 获取 web 容器的配置类，子容器（SpringMVC 配置文件）
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{AppConfig.class};
    }

    // 获取 DispatcherServlet 的映射信息
    // /：拦截所有请求（包括静态资源），不包括 *.jsp
    // /*：拦截所有请求包括 *.jsp，jsp 是 tomcat 的引擎解析的
    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }
}
```

```java
// Spring 容器不扫描 controller，父容器
@ComponentScan(value="com.demo", excludeFilters =
        {@ComponentScan.Filter(type= FilterType.ANNOTATION, classes = {Controller.class})}
)
public class RootConfig {
}
```

```java
// SpringMVC 只扫描 Controller，子容器
// useDefaultFilters = false：禁用默认排除规则
@ComponentScan(value="com.demo", includeFilters = {
    @ComponentScan.Filter(type= FilterType.ANNOTATION, classes = {Controller.class})
}, useDefaultFilters = false)
public class AppConfig {
}
```

## 6.异步请求

### 6.1 Servlet 3.0

```java
@WebServlet(value="/async", asyncSupported = true)
public class AsyncServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        // 支持异步处理：asyncSupported = true
        // 开启异步模式
        AsyncContext asyncContext = req.startAsync();

        // 进行异步处理
        asyncContext.start(new Runnable() {
            @Override
            public void run() {
                try {
                    sayHello();
                    asyncContext.complete();
                    // 获取异步上下文
                    AsyncContext asyncContext1 = req.getAsyncContext();
                    asyncContext1.getResponse().getWriter().write("hello async");
                } catch (InterruptedException | IOException e) {
                    e.printStackTrace();
                }
            }
        });

    }

    public void sayHello() throws InterruptedException {
        System.out.println(Thread.currentThread() + "running...");
        Thread.sleep(3000);
    }
}
```

### 6.2 SpringMVC

#### 1.Callable

1. 控制器返回 Callable
2. Spring 异步处理，将 Callable 提交到 TaskExecutor 使用一个隔离的线程进行执行
3. DispatcherServlet 和所有的 Filter 退出 web 容器的线程，但 response 保持打开状态
4. Callable 返回结果， SpringMVC 将请求重新再次派发给容器，恢复之前的处理
5. 根据 Callable 返回的结果，SpringMVC 继续进行视图渲染等（从收请求到最后处理）

```java
@Controller
public class AsyncController {
    @ResponseBody
    @RequestMapping("/async01")
    public Callable<String> async01() {
        System.out.println("main thread started" + Thread.currentThread());
        Callable c =  new Callable<String>(){
            @Override
            public String call() throws Exception {
                System.out.println("sub thread started" + Thread.currentThread());
                Thread.sleep(2000);
                System.out.println("sub thread end" + Thread.currentThread());
                return "Callable<String> async01";
            }
        };
        System.out.println("main thread end" + Thread.currentThread());
        return c;
    }
}
```

#### 2.异步拦截器

1. 原生 API： AsyncListener
2. SpringMVC：AsyncHandlerInterceptor

#### 3.DeferredResult

<img src="img/第4章_Servlet 3.0/image-20211226010650337-d1b93ba5a8783b866cf2eacc9a6566a2-e58b0a.png" alt="image-20211226010650337" style="zoom: 50%;" />

```java
// 先返回一个 DeferredResult<T> 对象，当调用该对象的 setResult() 方法时，才会返回给前端
@ResponseBody
@RequestMapping("/createOrder")
public DeferredResult<Object> createOrder() {
    DeferredResult<Object> objectDeferredResult = new DeferredResult<>((long)10000, "create failed");
    DeferredResultQueue.save(objectDeferredResult);
    return objectDeferredResult;
}

@ResponseBody
@RequestMapping("/create")
public String create() {
    String order = UUID.randomUUID().toString();
    DeferredResult<Object> objectDeferredResult = DeferredResultQueue.get();
    // 向 DeferredResult 中设置值
    objectDeferredResult.setResult(order);
    return "success" + order;
}

public class DeferredResultQueue {
    private static Queue<DeferredResult<Object>> queue = new ConcurrentLinkedDeque<>();
    public static void save(DeferredResult<Object> deferredResult) {
        queue.add(deferredResult);
    }
    public static DeferredResult<Object> get() {
        return queue.poll();
    }
}
```

