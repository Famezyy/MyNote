# Spring容器重要组件

## 1.AbstractApplicationContext

容器的创建方式有两种，一种通过注解加载 bean，此时使用的是`AnnotationConfigApplicationContext`；另一种是通过 xml 配置文件加载，此时使用的上下文是`ClassPathXmlApplicationContext`，不过他们最终都会调用父类`AbstractApplicationContext`的`refresh()`方法来启动容器。

**1.AnnotationConfigApplicationContext**

```java
public class AnnotationConfigApplicationContext {
    public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
        // ↓
        this();
        // 注册启动类的 BeanDefinition
        register(componentClasses);
        // 刷新容器
        refresh();
    }
    
    public AnnotationConfigApplicationContext() {
		StartupStep createAnnotatedBeanDefReader = this.getApplicationStartup().start("spring.context.annotated-bean-reader.create");
        // 注册以下组件的 BeanDefinition
        // ConfigurationClassPostProcessor
        // AutowiredAnnotationBeanPostProcessor
        // CommonAnnotationBeanPostProcessor
        // EventListenerMethodProcessor
        // DefaultEventListenerFactory
		this.reader = new AnnotatedBeanDefinitionReader(this);
		createAnnotatedBeanDefReader.end();
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}
}
```

`this()`

调用父类 GenericApplicationContext 的无参构造方法。

```java
public GenericApplicationContext() {
 // 创建一个 DefaultListableBeanFactory
 this.beanFactory = new DefaultListableBeanFactory();
}
```

继承关系

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220605014138550-a8f15186b1d58559db3046f1fa3d2524-dd0609.png" alt="image-20220605014138550" style="zoom:80%;" />

通过`SpringApplication.run()`方法启动时，流程稍有不同：

先根据类型创建容器：

- `ServletWebServerApplicationContext`
- `ReactiveWebServerApplicationContext`

再在重载的`run(String... args)`方法中执行`this.refreshContext(context)`来刷新容器：

```java
public class SpringApplication {
    private void refreshContext(ConfigurableApplicationContext context) {
        if (this.registerShutdownHook) {
            shutdownHook.registerApplicationContext(context);
        }
        // 刷新容器
        this.refresh(context);
    }
```

**2.ClassPathXmlApplicationContext**

```java
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
    this(new String[] {configLocation}, true, null);
}
public ClassPathXmlApplicationContext(
    String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
    throws BeansException {

    super(parent);
    setConfigLocations(configLocations);
    if (refresh) {
        refresh();
    }
}
```

继承关系

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220605013914031-2cb8f6c95473819a71d71c99768ef527-a384d3.png" alt="image-20220605013914031" style="zoom:80%;" />

```java
class ConfigurationClassBeanDefinitionReader {

    public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
        TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
        for (ConfigurationClass configClass : configurationModel) {
            // ↓
            loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
        }
    }

    // ↑
    private void loadBeanDefinitionsForConfigurationClass(
        ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

        if (trackedConditionEvaluator.shouldSkip(configClass)) {
            String beanName = configClass.getBeanName();
            if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
                this.registry.removeBeanDefinition(beanName);
            }
            this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
            return;
        }
        // 判断当前配置候选类是否由其他类 import 导入或者是否是某个配置类的 Bean
        // Import 导入的类在这里完成 Beandefinition 加载
        if (configClass.isImported()) {
            registerBeanDefinitionForImportedConfigurationClass(configClass);
        }
        // 调用 getBeanMethods() 获取所有存储的 BeanMethod 对象
        // 注册 @Bean 注解标注的 Bean 的 BeanDefinition！
        for (BeanMethod beanMethod : configClass.getBeanMethods()) {
            loadBeanDefinitionsForBeanMethod(beanMethod);
        }

        // 注册 ImportBeanDefinitionRegistrar 导入的 Beandefinition
        // 遍历 importBeanDefinitionRegistrars：Map
        loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
        loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
    }

}
```

## 2.BeanDefinition

保存了 Bean 的一系列定义信息的对象，例如全类名、属性名、懒加载等。

## 3.BeanFactoryPostProcessor

BeanFactory 的后置处理器，在 BeanFactory 标准初始化之后执行。

### BeanDefinitionRegistryPostProcessor

`BeanFactoryPostProcessor`的子接口，在所有 Bean 定义信息将要被加载，但 Bean 实例还未创建时调用（在`BeanFactoryPostProcessor`前执行）。如果是注解容器，则会通过`ConfigurationClassPostProcessor`加载`BeanDefinition`。

可利用 BeanDefinitionRegistryPostProcessor 给容器中额外添加组件。

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
    // BeanDefinitionRegistry：Bean 定义信息的保存中心，以后 BeanFactory 就是按照 BeanDefinitionRegistry 里保存的每个 Bean 的信息创建实例
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry var1) throws BeansException;
}
```

**例子**

```java
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        int beanDefinitionCount = configurableListableBeanFactory.getBeanDefinitionCount();
        String[] beanDefinitionNames = configurableListableBeanFactory.getBeanDefinitionNames();
    }

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
        // RootBeanDefinition beanDefinitionBuilder = new RootBeanDefinition(Red.class);
        BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.rootBeanDefinition(Red.class);
        beanDefinitionRegistry.registerBeanDefinition("hello", beanDefinitionBuilder.getBeanDefinition());
    }
}
```

## 4.MessageSource

做国际化功能，消息绑定，消息解析。取出国际化配置文件中的某个 Key 的值，能按照区域信息获取。

### ResourceBundleMessageSource应用

1. `I18nService`封装`MessageSource`类，按照需要新增方法或简化调用链

   ```java
   public class I18nService {
   
       private final MessageSource messageSource;
   
       public I18nService(MessageSource messageSource) {
           this.messageSource = messageSource;
       }
   
       public String getMessage(String msgKey, Object[] args, Locale locale) {
           return messageSource.getMessage(msgKey, args, locale);
       }
   
       public String getMessage(String msgKey, Locale locale) {
           return messageSource.getMessage(msgKey, null, locale);
       }
   }
   ```

2. 配置`I18nService`，主要是配置资源文件的 baseNames

   ```java
   // 只注册一个 I18nService（推荐）， 只有当存在 messages[spring.messages.basename].properties 文件时，MessageSourceAutoConfiguration 才会自动向容器中注册 MessageSourceProperties 和 ResourceBundleMessageSource
   @Bean
   public I18nService i18nService(MessageSource messageSource) {
       return new I18nService(messageSource);
   }
   
   ================================================================================
   // 或者手动向容器中注册一个 ResourceBundleMessageSource，此时 MessageSourceAutoConfiguration 不再工作
   @Bean
   public I18nService i18nService(ResourceBundleMessageSource resourceBundleMessageSource) {
       return new I18nService(resourceBundleMessageSource);
   }
   
   @Bean
   public ResourceBundleMessageSource messageSource() {
       Locale.setDefault(Locale.CHINESE); // 在 ResourceBundle 会中调用 Locale.getDefault() 获取默认的地区
       ResourceBundleMessageSource source = new ResourceBundleMessageSource();
       source.setBasenames("messages"); // name of the resource bundle
       source.setUseCodeAsDefaultMessage(true);
       source.setDefaultEncoding("UTF-8");
       return source;
   }
   
   =================================================================================
   // 或者往容器中加入 messageSourceProperties 后利用配置文件更改属性
   @Bean
   public I18nService i18nService(ResourceBundleMessageSource resourceBundleMessageSource) {
       return new I18nService(resourceBundleMessageSource);
   }
   
   @ConfigurationProperties(prefix = "spring.messages")
   @Bean
   public MessageSourceProperties messageSourceProperties() {
       return new MessageSourceProperties();
   }
   
   @Bean
   public ResourceBundleMessageSource messageSource(MessageSourceProperties messageSourceProperties) {
       Locale.setDefault(Locale.CHINESE);
       ResourceBundleMessageSource source = new ResourceBundleMessageSource();
       source.setBasename(messageSourceProperties.getBasename()); // basename 可同时设置多个，用 "," 分隔开
       source.setDefaultEncoding(String.valueOf(messageSourceProperties.getEncoding()));
       return source;
   }
   ```

3. 多语言资源配置文件（basenames = i18n/messages），即配置文件所在目录为`i18n`，文件前缀为`messages`，和语言之间以下划线隔开，下划线后的语言可任意设置，需同 Locale 对象中传入的一致（由`ResourceBundle.toBundleName()`负责解析）

   - messages.properties

     ```properties
     message.key.test=测试!
     message.key.hello=你好！{0}~
     ```

   - messages_en.properties

     ```properties
     message.key.test=test!
     message.key.hello=hello！{0}~
     ```

4. 用于测试用的 controller

   ```java
   @Controller
   @RequestMapping(value = "/api")
   public class HelloJavaCoderController {
   
       private final I18nService i18nService;
   
       public HelloJavaCoderController(I18nService i18nService) {
           this.i18nService = i18nService;
       }
   
       @GetMapping("/hello-coder")
       public ResponseEntity greeting(@RequestHeader("Accept-Language") String lan) {
           return ResponseEntity.ok(i18nService.getMessage("message.key.hello", new Object[]{"JavaCoder"}, new Locale(lan)));
       }
   
       @GetMapping("/test")
       public ResponseEntity test(@RequestHeader("Accept-Language") String lan) {
           return ResponseEntity.ok(i18nService.getMessage("message.key.test", new Locale(lan))));
       }
   
   }
   ```

5. 使用 Postman 进行 restful API 测试

   1. 请求 http://localhost:8888/api/test 并在 header 中设置 Accept-Languate=zh，结果如下：

      <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hhaWh1aV95YW5n,size_16,color_FFFFFF,t_70-d93b4a0cf44ae98b7af96e9bdbc9e249-8607c9" alt="img" style="zoom: 33%;" />

   2. 请求 http://localhost:8888/api/test 并在 header 中设置 Accept-Language=en，结果如下：

      <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hhaWh1aV95YW5n,size_16,color_FFFFFF,t_701-46c65ce69f4c10babdb13c457f6d00cd-dcf1da" alt="img" style="zoom:33%;" />

   3. 请求 http://localhost:8888/hello-coder 并在 header 中设置 Accept-Language=zh，结果如下：

      <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hhaWh1aV95YW5n,size_16,color_FFFFFF,t_702-a7a8df55433872b6e7951bb5de46a007-353c91" alt="img" style="zoom:33%;" />

   4. 请求 http://localhost:8888/hello-coder 并在 header 中设置 Accept-Languate=en，结果如下：

      <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hhaWh1aV95YW5n,size_16,color_FFFFFF,t_703-83916d5e4945286e1cdf98e88e6ef315-3af9c6" alt="img" style="zoom:33%;" />

## 5.ApplicationListener

监听容器中发布的事件，事件驱动模型开发，用来监听`ApplicationEvent`及其下面的子事件。

**代码实现**

写一个监听器（`ApplicationListener`的实现类）来监听某个事件（`ApplicationEvent`及其下面的子类），把监听器放在容器中。

```java
@Component
public class MyApplicationListener implements ApplicationListener {
    // 当容器中发布此事件时触发
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        System.out.println("收到事件：" + event);
    }
}
```

只要容器中有相关时间的发布，我们就能监听到事件，常见的默认事件有：

- `ContextRefreshedEvent`：容器刷新完成（所有 Bean 都完全创建）会发布这个事件

  ```java
  class AbstractApplicationContext {
  
      protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
          Assert.notNull(event, "Event must not be null");
  
          // Decorate event as an ApplicationEvent if necessary
          ApplicationEvent applicationEvent;
          if (event instanceof ApplicationEvent) {
              applicationEvent = (ApplicationEvent) event;
          }
          else {
              applicationEvent = new PayloadApplicationEvent<>(this, event);
              if (eventType == null) {
                  eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
              }
          }
  
          // Multicast right now if possible - or lazily once the multicaster is initialized
          if (this.earlyApplicationEvents != null) {
              this.earlyApplicationEvents.add(applicationEvent);
          }
          else {
              // 获取事件的多播器（派发器）
              // 调用 multicastEvent() 派发事件
              getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
          }
  
          // Publish event via parent context as well...
          if (this.parent != null) {
              if (this.parent instanceof AbstractApplicationContext) {
                  ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
              }
              else {
                  this.parent.publishEvent(event);
              }
          }
      }
  }
  ```

  **派发器原理**

  ```java
  class SimpleApplicationEventMulticaster {
      @Override
      public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
          ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
          Executor executor = getTaskExecutor();
          // 获取到所有的 applicationListener
          for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
              // 如果有 executor，可以支持使用 executor 进行异步派发
              if (executor != null) {
                  // 拿到 listener 回调 onapplicationEvent() 方法
                  executor.execute(() -> invokeListener(listener, event));
              }
              else {
                  invokeListener(listener, event);
              }
          }
      }
  }
  ```

- `ContextClosedEvent`：关闭容器会发布这个事件

手动发布一个事件：

```java
@Test
public void test01() {
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(TxConfig.class);
    // publishEvent() 方法用来发布事件
    applicationContext.publishEvent(new ApplicationEvent(new String("my event")) {
    });

    applicationContext.close();
}
```

### 原理

1. 事件多播器（派发器）

   `refresh()` -- `initApplicationEventMulticaster()`

   - 获取 beanFactory
   - 从 beanFactory 中寻找 id 为`applicationEventMulticaster`，类型为`ApplicationEventMulticaster`的组件
   - 如果没有，就 new 一个`SimpleApplicationEventMulticaster`放到 beanFactory 中，其他组件派发事件时，自动注入`applicationEventMulticaster`

   > 默认同步执行潜艇其，相当于`org.springframework.core.task.SyncTaskExecutor`。
   >
   > 可以指定一个异步任务执行器（`AsyncTaskExecutor`），不阻塞调用者。但是，请注意异步执行不会参与调用者的线程上下文（类加载器、事务关联），除非`TaskExecutor`支持。

2. 监听器的加载

   `refresh()` -- `registerListeners()`

   - 从容器中获取所有的监听器名称，并注册到 ApplicationEventMulticaster 中

### @EventListener

注解在任意方法上即可实现监听器的功能。

```java
@Service
public class UserService {
    @EventListener(classes = {ApplicationEvent.class})
    public void listen(ApplicationEvent event) {
        System.out.println("userService 监听：" + event);
    }
}
```

**原理**

【调用`SmartInitializingSingleton`的`afterSingletonsInstantiated()`】时，由`EventListenerMethodProcessor`来解析方法并创建`applicationListener`对象。

## 6.Application Events and Listeners

```java
public class MyApplicationContextInitializer implements ApplicationContextInitializer {
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        System.out.println("initializing..");
    }
}
```

```java
public class MyApplicationListener implements ApplicationListener {
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        System.out.println("reveived event:" + event.getSource());
    }
}
```

```java
public class MySpringApplicationRunListener implements SpringApplicationRunListener {

    private SpringApplication springApplication;
    private String[] args;

    // 需要一个构造参数
    public MySpringApplicationRunListener(SpringApplication springApplication, String[] args) {
        this.springApplication = springApplication;
        this.args = args;
    }

    @Override
    public void starting(ConfigurableBootstrapContext bootstrapContext) {
        System.out.println("start");
        SpringApplicationRunListener.super.starting(bootstrapContext);
    }

    @Override
    public void environmentPrepared(ConfigurableBootstrapContext bootstrapContext, ConfigurableEnvironment environment) {
        System.out.println("environmentPrepared");
        SpringApplicationRunListener.super.environmentPrepared(bootstrapContext, environment);
    }

    @Override
    public void contextPrepared(ConfigurableApplicationContext context) {
        System.out.println("contextPrepared");
        SpringApplicationRunListener.super.contextPrepared(context);
    }

    @Override
    public void contextLoaded(ConfigurableApplicationContext context) {
        System.out.println("contextPrepared");
        SpringApplicationRunListener.super.contextLoaded(context);
    }

    @Override
    public void started(ConfigurableApplicationContext context, Duration timeTaken) {
        System.out.println("started");
        SpringApplicationRunListener.super.started(context, timeTaken);
    }

    @Override
    public void ready(ConfigurableApplicationContext context, Duration timeTaken) {
        System.out.println("ready");
        SpringApplicationRunListener.super.ready(context, timeTaken);
    }

    @Override
    public void failed(ConfigurableApplicationContext context, Throwable exception) {
        System.out.println("failed");
        SpringApplicationRunListener.super.failed(context, exception);
    }
}
```

在`META-INF/spring.factories`中配置以上三个监听器：

```bash
org.springframework.boot.SpringApplicationRunListener=com.youyi.hello.MySpringApplicationRunListener
org.springframework.context.ApplicationListener=com.youyi.hello.MyApplicationListener
org.springframework.context.ApplicationContextInitializer=com.youyi.hello.MyApplicationContextInitializer
```

## 7.CommandLineRunner

```java
@Order(2) // 数字越大优先级越高
@Component
public class MyCommandLineRunner implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {
        System.out.println("String");
    }
}
```

## 8.ApplicationRunner

```java
@Order(2)
@Component
public class MyApplicationRunner implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) {
        System.out.println("ApplicationArguments");
    }

}
```

