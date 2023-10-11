# 第4章_Servlet 3.0

## 1.Servlet 3.0

### 1.1 快速开始

**@WebServlet**

```java
@WebServlet(name = "helloServlet", value = "/hello-servlet")
public class HelloServlet extends HttpServlet {
    private String message;

    public void init() {
        message = "Hello World!";
    }

    public void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        response.setContentType("text/html");

        // Hello
        PrintWriter out = response.getWriter();
        out.println("<html><body>");
        out.println("<h1>" + message + "</h1>");
        out.println("</body></html>");
    }

    public void destroy() {
    }
}
```

```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<!DOCTYPE html>
<html>
<head>
    <title>JSP - Hello World</title>
</head>
<body>
<h1><%= "Hello World!" %>
</h1>
<br/>
<a href="hello-servlet">Hello Servlet</a>
</body>
</html>
```

### 1.2 Shared libraries（共享库）& runtimes pluggability（运行时插件）

1. Servlet 容器启动时，会扫描当前应用中的每一个 jar 包的 ServletContainerInitializer 的实现

2. 提供**ServletContainerInitializer**的实现类

   ```java
   // 容器启动时会将 HandlesTypes 指定类型的下面的子类，实现类，子接口等全部传递过来
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

3. 绑定在**META-INF/services/javax.servlet.ServletContainerInitializer**，文件的内容就是 ServletContainerInitializer 的实现类的全类名（Idea 要放在 resources 路径下）

   ```java
   com.example.servlet.MyServletContainerInitializer
   ```

**总结**

容器在启动应用时，会扫描当前应用每一个 jar 包里面的 META-INF/Services/javax.servlet.ServletContainerInitializer 指定的实现类，启动并运行这个实现类方法

### 1.3 使用ServletContext注册web组件（Servlet，Filter，Listener）

1. 使用 ServletContext 注册 web 组件（Servlet，Filter，Listener）

   ```java
   public void onStartup(Set<Class<?>> set, ServletContext servletContext) throws ServletException {
       // 注册组件
       ServletRegistration.Dynamic userServlet = servletContext.addServlet("userServlet", new UserServlet());
       // 配置 Servlet 映射信息
       userServlet.addMapping("/user");
   
       // 注册监听器
       servletContext.addListener(UserListener.class);
   
       // 注册过滤器
       FilterRegistration.Dynamic userFilter = servletContext.addFilter("UserFilter", UserFilter.class);
       // 配置 Filter 映射信息
       userFilter.addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST), true, "/*");
   }
   ```

2. 使用编码的方式，在项目启动时给 ServletContext 里添加组件，必须在项目启动时添加

   1. ServletContainerInitializer 得到 ServletContext

   2. ServletContextListener 得到 ServletContext

      ```java
      public void contextInitialized(ServletContextEvent sce) {
          ServletContext servletContext = sce.getServletContext();
          System.out.println("context initialized");
      }
      ```

### 1.4 与SpringMVC整合

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

### 1.5 定制SpringMVC

1. `@EnableWebMvc`：开启 SpringMVC 定制配置功能，相当于`<mvc:annotation-driven/>`

2. 配置组件（视图解析器，视图映射，静态资源映射，拦截器）：实现`WebMvcConfigurer`接口，重写方法

   ```java
   @ComponentScan(value="com.demo", includeFilters = {
           @ComponentScan.Filter(type= FilterType.ANNOTATION, classes = {Controller.class})
   }, useDefaultFilters = false)
   @EnableWebMvc
   public class AppConfig implements WebMvcConfigurer {
   
       // 定制视图解析器
       @Override
       public void configureViewResolvers(ViewResolverRegistry registry) {
           // 默认所有页面都从 /WEB-INF/xxx.jsp
           // registry.jsp();
           registry.jsp("/WEB-INF/views", ".jsp");
       }
   
       // 静态资源访问
       @Override
       public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
           configurer.enable();
       }
   
       // 拦截器
       @Override
       public void addInterceptors(InterceptorRegistry registry) {
           registry.addInterceptor(new HandlerInterceptor() {
   
               // 目标方法前执行
               @Override
               public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
                   return HandlerInterceptor.super.preHandle(request, response, handler);
               }
   
               // 目标方法后执行
               @Override
               public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
                   HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);
               }
   
               // 页面响应后执行
               @Override
               public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
                   HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
               }
               // /** 拦截所有请求：与过滤器不同
           }).addPathPatterns("/**");
       }
   }
   ```

### 1.6 异步请求

#### 1.Servlet 3.0

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

#### 2.SpringMVC

##### 2.1 Callable

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

##### 2.2 异步拦截器

1. 原生 API： AsyncListener
2. SpringMVC：AsyncHandlerInterceptor

##### 2.3 DeferredResult

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20211226010650337-d1b93ba5a8783b866cf2eacc9a6566a2-e58b0a.png" alt="image-20211226010650337" style="zoom: 50%;" />

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

