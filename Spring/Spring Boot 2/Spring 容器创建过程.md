# Spring容器创建过程

容器的创建方式有两种，一种通过注解加载 bean，此时使用的是`AnnotationConfigApplicationContext`；另一种是通过 xml 配置文件加载，此时使用的上下文是`ClassPathXmlApplicationContext`，不过他们最终都会调用父类`AbstractApplicationContext`的`refresh()`方法来启动容器。

**1.AnnotationConfigApplicationContext**

```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
    this();
    register(componentClasses);
    // 刷新容器
    refresh();
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

![image-20220605014138550](img/image-20220605014138550.png)

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

![image-20220605013914031](img/image-20220605013914031.png)

**refresh()**

```java
public abstract class AbstractApplicationContext {
    @Override
    public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {
            StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

            // Prepare this context for refreshing.
            prepareRefresh();

            // Tell the subclass to refresh the internal bean factory.
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            // Prepare the bean factory for use in this context.
            prepareBeanFactory(beanFactory);

            try {
                // Allows post-processing of the bean factory in context subclasses.
                postProcessBeanFactory(beanFactory);

                StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
                // Invoke factory processors registered as beans in the context.
                invokeBeanFactoryPostProcessors(beanFactory);

                // Register bean processors that intercept bean creation.
                registerBeanPostProcessors(beanFactory);
                beanPostProcess.end();

                // Initialize message source for this context.
                initMessageSource();

                // Initialize event multicaster for this context.
                initApplicationEventMulticaster();

                // Initialize other special beans in specific context subclasses.
                onRefresh();

                // Check for listener beans and register them.
                registerListeners();

                // Instantiate all remaining (non-lazy-init) singletons.
                finishBeanFactoryInitialization(beanFactory);

                // Last step: publish corresponding event.
                finishRefresh();
            }

            catch (BeansException ex) {
                if (logger.isWarnEnabled()) {
                    logger.warn("Exception encountered during context initialization - " +
                                "cancelling refresh attempt: " + ex);
                }
                // Destroy already created singletons to avoid dangling resources.
                destroyBeans();
                // Reset 'active' flag.
                cancelRefresh(ex);
                // Propagate exception to caller.
                throw ex;
            }

            finally {
                // Reset common introspection caches in Spring's core, since we
                // might not ever need metadata for singleton beans anymore...
                resetCommonCaches();
                contextRefresh.end();
            }
        }
    }
}
```

## 1.刷新前的预处理

`prepareRefresh()` 

1. `initPropertySources()`初始化一些属性设置，子类自定义个性化的属性设置方法
2. `getEnvironment().validateRequiredProperties()`检验属性
3. `earlyApplicationListeners`保存其中一些早期的事件

## 2.创建BeanFactory

`ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory()` 

```java
public abstract class AbstractApplicationContext {
    protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        // 调用子类的 refreshBeanFactory() 刷新 BeanFactoty
        refreshBeanFactory();
        // 返回 BeanFactory 对象，DefaultListableBeanFactory 类型
        return getBeanFactory();
    }
}
```

### 2.1 注解容器

> 注解启动时，所有的用户`BeanDefinition`在执行`invokeBeanFactoryPostProcessors(beanFactory)`才被加载。

```java
public class GenericApplicationContext {
    protected final void refreshBeanFactory() throws IllegalStateException {
        if (!this.refreshed.compareAndSet(false, true)) {
            throw new IllegalStateException(
                "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
        }
        // 为 BeanFacotry 设置 ID
        this.beanFactory.setSerializationId(getId());
    }
}
```

### 2.2 XML容器

在`loadBeanDefinitions(beanDefinitionReader)`中，最终调用`XmlBeanDefinitionReader`来**解析** XML 文件并调用`DefaultListableBeanFactory`来**注册**所有的 BeanDefinition，并存放到`beanDefinitionMap`中。

```java
public abstract class AbstractXmlApplicationContext {
    protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
        // Create a new XmlBeanDefinitionReader for the given BeanFactory.
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

        // Configure the bean definition reader with this context's
        // resource loading environment.
        beanDefinitionReader.setEnvironment(this.getEnvironment());
        beanDefinitionReader.setResourceLoader(this);
        beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

        // Allow a subclass to provide custom initialization of the reader,
        // then proceed with actually loading the bean definitions.
        initBeanDefinitionReader(beanDefinitionReader);
        // 加载所有的 BeanDefinition
        loadBeanDefinitions(beanDefinitionReader);
    }
}
```

> **BeanDefinition**
>
> 保存了 Bean 的一系列定义信息的对象，例如全类名、属性名、懒加载等。

## 3.BeanFactory预准备工作

`prepareBeanFactory(beanFactory)` （对 BeanFactory 进行一些设置）

1. 设置 BeanFactory 的类加载器，支持表达式解析器
2. 添加部分 BeanPostProcessor（例：[引用：ApplicationContextAwareProcessor](Spring Boot 2 注解驱动开发.md#ApplicationContextAwareProcessor)）
3. 设置忽略的自动装配的接口 EnvironmentAware，EmbeddedValueResolverAware 等，该接口实现类不能通过接口类型自动注入
4. 注册可以解析的自动装配，可以直接在任何组件中自动注入 BeanFactory，ResourceLoader，ApplicationEventPublisher，ApplicationContext
5. 添加 BeanPostProcessor：`ApplicationListenerDetector`
6. 添加编译时的 AspectJ
7. 添加一些组件
   - environment：`ConfigurableEnvironment`

   - SystemProperties：Map<String, Object>

   - SystemEnvironment：Map<String, Object>

## 4.BeanFactory准备工作完成后进行的后置处理工作

`postProcessBeanFactory(beanFactory)` （本类方法，未写实现体）

> 子类通过重写这个方法在 BeanFactory 创建并预准备完成以后做进一步设置

## ==5.执行BeanFactoryPostProcessor==

```java
public abstract class AbstractApplicationContext {
    protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        // 执行 BeanFactory 的后置处理器
        PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
		...
    }
}
```

- `BeanFactoryPostProcessor`：BeanFactory 的后置处理器，在 BeanFactory**标准初始化之后**执行

- 两个接口 [BeanFactoryPostProcessor](Spring Boot 2 注解驱动开发.md#BeanFactoryPostProcessor)，[BeanDefinitionRegistryPostProcessor](Spring Boot 2 注解驱动开发.md#BeanDefinitionRegistryPostProcessor)，后者继承了前者。

- **执行过程**

  1. 执行**BeanDefinitionRegistryPostProcessor**的方法`postProcessBeanDefinitionRegistry(registry)`

     1. 获取所有的`BeanDefinitionRegistryPostProcessor`

     2. 先执行实现了**PriorityOrdered**接口的 BeanDefinitionRegistryPostProcessor

        > **加载 BeanDefinition（解析配置类一步一步加载所有其他的）**
        >
        > 注解启动时，通过**ConfigurationClassPostProcessor**在这里加载了所有的`BeanDefinition`，该类实现了`BeanDefinitionRegistryPostProcessor`、`PriorityOrdered`接口。
        >
        > ```java
        > public class ConfigurationClassPostProcessor {
        >     public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
        >         do {
        >             StartupStep processConfig = this.applicationStartup.start("spring.context.config-classes.parse");
        >             // 1.解析配置候选类，注册其 BeanDefinition
        >             // 2.将配置候选类的 MetaData、resource 等信息存放在缓存 configurationClasses 中
        >             // 配置候选类是标注了 Confiugration、Component、Serivce、Controller 等注解的类的 BeandefinitionHolder
        >             // 不包括配置类中的 Bean 注解标注的类
        >             parser.parse(candidates); // ↓
        >             parser.validate();
        >             // 3.从 configurationClasses 中获取所有的配置候选类
        >             Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
        >             configClasses.removeAll(alreadyParsed);
        > 
        >             if (this.reader == null) {
        >                 this.reader = new ConfigurationClassBeanDefinitionReader(
        >                     registry, this.sourceExtractor, this.resourceLoader, this.environment,
        >                     this.importBeanNameGenerator, parser.getImportRegistry());
        >             }
        >             // 4.加载注册其他 BeanDefinitions
        >             this.reader.loadBeanDefinitions(configClasses);
        >             ...
        >         }
        >     }
        >     while (!candidates.isEmpty());
        > }
        > 
        > // 解析配置类
        > public void parse(Set<BeanDefinitionHolder> configCandidates) {
        >     for (BeanDefinitionHolder holder : configCandidates) {
        >         BeanDefinition bd = holder.getBeanDefinition();
        >         // 里面调用 ConfigurationClassParser 的 processConfigurationClass()
        >         parse(bd.getBeanClassName(), holder.getBeanName());
        >     }
        >     this.deferredImportSelectorHandler.process();
        > }
        > ```
        >
        > 1. 解析**配置候选类**并注册其`BeanDefinition`。
        >
        >    ```java
        >    class ConfigurationClassParser {
        >        protected void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter) throws IOException {
        >            do {
        >                // 1.解析配置候选类
        >                sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
        >            } while (sourceClass != null);
        >            // 2.将配置候选类的 MetaData、resource 等信息存放在缓存 configurationClasses 中
        >            this.configurationClasses.put(configClass, configClass);
        >        }
        >    }
        >    ```
        >
        >    ```java
        >    class ConfigurationClassParser {
        >        protected final SourceClass doProcessConfigurationClass() {
        >            // 1.1 解析 @Component
        >            if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
        >                // 解析内部类
        >    			processMemberClasses(configClass, sourceClass, filter);
        >    		}
        >            // 1.2 解析 @PropertySource
        >            // 1.3 解析 @ComponentScan
        >            Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
        >                sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
        >            if (!componentScans.isEmpty() &&
        >                !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        >                for (AnnotationAttributes componentScan : componentScans) {
        >                    // 执行扫描并注册 BeanDefinition
        >                    Set<BeanDefinitionHolder> scannedBeanDefinitions =
        >                        this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
        >                    // 解析所有扫描到的配置类
        >                    for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
        >                        BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
        >                        if (bdCand == null) {
        >                            bdCand = holder.getBeanDefinition();
        >                        }
        >                        // 是否是 configuration 的候选类
        >                        if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
        >                            // 是则调用 parse 来解析
        >                            parse(bdCand.getBeanClassName(), holder.getBeanName()); // ↓
        >                        }
        >                    }
        >                }
        >            }
        >            // 1.4 解析 @Import
        >            	// 递归判断是否有 @Import 注解，将有该注解的所有 sourceClass 存入 set
        >            	// 解析 sourceClassSet
        >                	//if -> 判断是否是 ImportSelector 的子实现类
        >            			//yes -> 调用自己来解析 ImportSelector 导入的类（如果是普通的 Bean，则会执行最后）
        >            		//if ->判断是否是 ImportBeanDefinitionRegistrar 的子实现类，是的话存入 importBeanDefinitionRegistrars map 中
        >            		//else -> 按照配置类处理，将其封装为 ConfigurationClass，将导入该类的 configClass 加入 importedBy
        >            		//		  而后调用最开始的 processConfigurationClass()
        >            
        >            // 1.5 Process any @ImportResource annotations
        >            
        >            // 1.6 解析 BeanMethod，将其添加到封装了配置候选类的 SourceClass 中
        >            configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
        >            // others
        >        }
        >        protected final void parse(@Nullable String className, String beanName) throws IOException {
        >    		// 获取文件的元数据
        >    		MetadataReader reader = this.metadataReaderFactory.getMetadataReader(className);
        >            // 当成配置类来嵌套解析
        >    		processConfigurationClass(new ConfigurationClass(reader, beanName), DEFAULT_EXCLUSION_FILTER);
        >    	}
        >    }
        >    ```
        >
        >    执行扫描：
        >
        >    ```java
        >    class ComponentScanAnnotationParser {
        >        public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, String declaringClass) {
        >            Set<String> basePackages = new LinkedHashSet<>();
        >            // 1.1 解析 ComponentScan 的 basePackages 属性
        >            String[] basePackagesArray = componentScan.getStringArray("basePackages");
        >            for (String pkg : basePackagesArray) {
        >                String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
        >                                                                       ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
        >                Collections.addAll(basePackages, tokenized);
        >            }
        >            for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
        >                basePackages.add(ClassUtils.getPackageName(clazz));
        >            }
        >            // 1.2 如果 basePackages 属性为空，则设置为当前配置候选类所在包
        >            if (basePackages.isEmpty()) {
        >                basePackages.add(ClassUtils.getPackageName(declaringClass));
        >            }
        >            // 1.3 扫描 basePackages
        >            return scanner.doScan(StringUtils.toStringArray(basePackages)); // ↓
        >        }
        >    }
        >    public class ClassPathBeanDefinitionScanner {
        >        protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        >            Assert.notEmpty(basePackages, "At least one base package must be specified");
        >            Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
        >            for (String basePackage : basePackages) {
        >                // 扫描路径下每一个目录里的 class 文件
        >                // 最终会调用 PathMatchingResourcePatternResolver 的 doRetrieveMatchingFiles()
        >                Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        >    
        >                for (BeanDefinition candidate : candidates) {
        >                    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
        >                    candidate.setScope(scopeMetadata.getScopeName());
        >                    String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
        >                    if (candidate instanceof AbstractBeanDefinition) {
        >                        postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
        >                    }
        >                    if (candidate instanceof AnnotatedBeanDefinition) {
        >                        AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
        >                    }
        >                    if (checkCandidate(beanName, candidate)) {
        >                        BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
        >                        definitionHolder =
        >                            AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
        >                        beanDefinitions.add(definitionHolder);
        >                        // 1.4 最终调用 DefaultListableBeanFactory 的 registerBeanDefinition() 注册 BeanDefinition
        >                        registerBeanDefinition(definitionHolder, this.registry);
        >                    }
        >                }
        >            }
        >            return beanDefinitions;
        >        }
        >    }
        >    ```
        >
        > 2. 将**配置候选类**的 MetaData、resource 等信息存放在配置缓存`configurationClasses`中
        >
        > 3. 从 configurationClasses 中获取所有的配置候选类
        >
        > 4. 加载注册其他 BeanDefinitions
        >
        >    例如，被`@import`导入的 Bean、`ImportBeanDefinitionRegistrar`中导入的 Bean、`@Bean`修饰的 Bean 等的`BeanDefinition`。
        >
        >    底层就是放入`beanDefinitionMap`中。
        >
        >    ```java
        >    class ConfigurationClassBeanDefinitionReader {
        >        public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
        >    		TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
        >    		for (ConfigurationClass configClass : configurationModel) {
        >    			loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator); // ↓
        >    		}
        >    	}
        >    
        >        private void loadBeanDefinitionsForConfigurationClass(
        >    			ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {
        >    
        >            // other operation
        >            // 判断当前配置候选类是否由其他类 import 导入或者是否是某个配置类的 Bean
        >            // Import 导入的类在这里完成 Beandefinition 加载
        >    		if (configClass.isImported()) {
        >    			registerBeanDefinitionForImportedConfigurationClass(configClass);
        >    		}
        >            // 调用 getBeanMethods 获取所有存储的 BeanMethod 对象
        >            // 注册 @Bean 注解标注的 Bean 的 BeanDefinition！
        >    		for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        >    			loadBeanDefinitionsForBeanMethod(beanMethod);
        >    		}
        >    		loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
        >            // 注册 ImportBeanDefinitionRegistrar 导入的 Beandefinition（在解析 Import 注解时会检查该类是否是 ImportBeanDefinitionRegistrar -> 1.3）
        >            // 遍历 importBeanDefinitionRegistrars map
        >    		loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
        >    	}
        >    
        >    }
        >    ```
        >
        > 最终在`refresh()`中执行`finishBeanFactoryInitialization()`时加载所有的 bean
  
     3. 再执行实现了**Ordered**接口的 BeanDefinitionRegistryPostProcessor
  
     4. 最后执行没有实现任何优先级或者顺序接口的 BeanDefinitionRegistryPostProcessor
  
  2. 再执行**BeanFactoryPostProcessor**的方法
  
     - 过程同`BeanDefinitionRegistryPostProcessor`

## 6.注册BeanPostProcessor

`registerBeanPostProcessors(beanFactory)` （Bean 的后置处理器：拦截 Bean 的创建过程）

- 五个接口：不同类型的 BeanPostProcessor 执行时机不同

  - BeanPostProcessor
  - DestructionAwareBeanPostProcessor
  - [引用：InstantiationAwareBeanPostProcessor（AOP）](Spring Boot 2 注解驱动开发.md#InstantiationAwareBeanPostProcessor)
  - SmartInstantiationAwareBeanPostProcessor
  - [引用：MergedBeanDefinitionPostProcessor](Spring Boot 2 注解驱动开发.md#MergedBeanDefinitionPostProcessor)：会被放到 internalPostProcessors 集合中

- **执行过程**

1. 获取所有的 BeanPostProcessor，后置处理器都可以默认通过 PriorityOrdered 和 Ordered 接口来指定优先级
  2. 先注册 PriorityOrdered 优先级接口的 BeanPostProcessor，把每一个 BeanPostProcessor 添加到 beanFactory 中

3. 再注册实现了 Ordered 接口的
  4. 最后注册没有实现任何优先级或者顺序接口的
  5. 最终注册 internalPostProcessors
  6. 注册一个 ApplicationListenerDetector，来在 Bean 创建实例完成后检查是否是 ApplicationListener，如果是则添加到容器中

## 7.初始化MessageSource组件

`initMessageSource()` （做国际化功能，消息绑定，消息解析）

1. 获取 beanFactory

2. 看容器中是否有 ID 为 MessageSource 类型为 MessageSource 的组件

   - 如果有，则赋值给 this.messageSource
     - MessageSource：取出国际化配置文件中的某个 Key 的值，能按照区域信息获取
   - 如果没有，则自己创建一个 DelegatingMessageSource，但需要手动设置 setParentMessageSource()，否则无法使用

3. 把创建好的 MessageSource 注册到容器中，以后获取国际化配置文件中的值时，可以自动注入 MessageSource，调用 getMessage() 方法

4. 应用（**ResourceBundleMessageSource**）：

   1. I18nService 封装 MessageSource 类，按照需要新增方法或简化调用链

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

   2. 配置 I18nService，主要是配置资源文件的 baseNames

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

   3. 多语言资源配置文件（basenames = i18n/messages），即配置文件所在目录为 i18n，文件前缀为 messages，和语言之间以下划线隔开，下划线后的语言可任意设置，需同 Locale 对象中传入的一致（由 ResourceBundle.toBundleName() 负责解析）

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
              return ResponseEntity.ok(i18nService.getMessage("message.key.hello", new Object[]{"JavaCoder"},new Locale(lan)));
          }
      
          @GetMapping("/test")
          public ResponseEntity test(@RequestHeader("Accept-Language") String lan) {
              return ResponseEntity.ok(i18nService.getMessage("message.key.test", new Locale(lan))));
          }
      
      }
      ```

   5. 使用 Postman 进行 restful API 测试

      1. 请求 http://localhost:8888/api/test 并在 header 中设置 Accept-Languate=zh，结果如下：

         <img src="img/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hhaWh1aV95YW5n,size_16,color_FFFFFF,t_70" alt="img" style="zoom: 33%;" />

      2. 请求 http://localhost:8888/api/test 并在 header 中设置 Accept-Language=en，结果如下：

         <img src="img/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hhaWh1aV95YW5n,size_16,color_FFFFFF,t_701" alt="img" style="zoom:33%;" />

      3. 请求 http://localhost:8888/hello-coder 并在 header 中设置 Accept-Language=zh，结果如下：

         <img src="img/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hhaWh1aV95YW5n,size_16,color_FFFFFF,t_702" alt="img" style="zoom:33%;" />

      4. 请求 http://localhost:8888/hello-coder 并在 header 中设置 Accept-Languate=en，结果如下：

         <img src="img/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hhaWh1aV95YW5n,size_16,color_FFFFFF,t_703" alt="img" style="zoom:33%;" />

## 8.初始化事件派发器

[引用：initApplicationEventMulticaster()](Spring Boot 2 注解驱动开发.md#ApplicationEventMulticaster)

## 9.初始化其他特殊Bean

`onRefresh()`

> 留给子容器（子类），重写该方法，在容器刷新时可以自定义逻辑

## 10.注册ApplicationListener

[引用：registerListeners()](Spring Boot 2 注解驱动开发.md#registerListeners)

## ==11.初始化所有剩下的单实例Bean==

```java
public abstract class AbstractApplicationContext {
    
    public void refresh() throws BeansException, IllegalStateException {
        ...
        // 实例化所有非懒加载的单例 bean
        finishBeanFactoryInitialization(beanFactory);
        ...
    }
    protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
        ...
        // 实例化所有非懒加载的单例 bean
        beanFactory.preInstantiateSingletons();
    }
}
```

最终调用`DefaultListableBeanFactory`的`preInstantiateSingletons()`方法

```java
public class DefaultListableBeanFactory {
    
    public void preInstantiateSingletons() throws BeansException {
        ...

        // 1.获取容器中的所有 Bean 名称
        List<String> beanNames = new ArrayList(this.beanDefinitionNames);
        Iterator var2 = beanNames.iterator();

        while(true) {
            String beanName;
            Object bean;
            do {
                while(true) {
                    RootBeanDefinition bd;
                    do {
                        do {
                            do {
                                // 4.当遍历创建完 Bean 后，再次重新遍历检查是否是 SmartInitializingSingleton，如果是则执行 afterSingletonsInstantiated()
                                if (!var2.hasNext()) {
                                    var2 = beanNames.iterator();

                                    while(var2.hasNext()) {
                                        beanName = (String)var2.next();
                                        Object singletonInstance = this.getSingleton(beanName);
                                        if (singletonInstance instanceof SmartInitializingSingleton) {
                                            StartupStep smartInitialize = this.getApplicationStartup().start("spring.beans.smart-initialize").tag("beanName", beanName);
                                            SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton)singletonInstance;
                                            if (System.getSecurityManager() != null) {
                                                AccessController.doPrivileged(() -> {
                                                    smartSingleton.afterSingletonsInstantiated();
                                                    return null;
                                                }, this.getAccessControlContext());
                                            } else {
                                                smartSingleton.afterSingletonsInstantiated();
                                            }

                                            smartInitialize.end();
                                        }
                                    }

                                    return;
                                }
                                beanName = (String)var2.next();
                                
                                // 2.获取 Bean 的定义信息：RootBeanDefinition
                                bd = this.getMergedLocalBeanDefinition(beanName);
                                
                                // 不是抽象的
                            } while(bd.isAbstract());
                            // 是单实例的
                        } while(!bd.isSingleton());
                        // 是懒加载的
                    } while(bd.isLazyInit());
                    
                    // 3.创建 Bean
					// 判断是否是 FactoryBean
                    if (this.isFactoryBean(beanName)) {
                        bean = this.getBean("&" + beanName);
                        break;
                    }
					// 不是 FactoryBean 则通过 getBean() 创建 bean！
                    this.getBean(beanName);
                }
            } while(!(bean instanceof FactoryBean));
            ...
        }
    }
}
```

### 1.【获取容器中的所有Bean名称】

### 2.【获取Bean的定义信息】

`RootBeanDefinition`

### 3.【创建Bean】

对于所有的**不是抽象的**、**单实例的**，**懒加载的** Bean，执行创建。

<a href="Spring 创建Bean.md">参考</a>

### 4.【执行`afterSingletonsInstantiated()】`

所有 Bean 创建完成后，在重新检查所有创建好的 Bean 是否是 SmartInitializingSingleton接口，如果是则执行`afterSingletonsInstantiated()`方法。

在 <a href="Spring Boot 2 注解驱动开发.md#EventListener">@EventListener</a> 中使用到了该方法。

## 12.完成容器初始化创建工作

`finishRefresh()`

1. 【清楚容器级别的缓存】：`clearResourceCaches()`
2. 【初始化和生命周期有关的后置处理器】：`initLifecycleProcessor()`
   - 默认从容器中找是否有 LifecycleProcessor 类型的组件
   - 如果没有则 new 一个放到容器中
   - 可以写一个 LifecycleProcessor 的实现类，可以在 BeanFacoty 刷新完成及关闭的时候调用
3. 【调用生命周期处理器（BeanFactory）的 onRefresh() 方法】：`getLifecycleProcessor().onRefresh()`
4. 【发布刷新完成事件】：`publishEvent(new ContextRefreshedEvent(this))`
5. `LiveBeansView.registerApplicationContext(this)`

## 总结

1. Spring 容器启动时，先会保存所有注册进来的 Bean 的定义信息

   1. xml 注册 Bean
   2. 注解注册 Bean

2. Spring 容器会在合适时机创建这些 Bean

   非懒加载的单例 Bean 会在容器创建时创建，会调用`BeanFactory`子实现类的`doGetBean()`来创建，创建好后保存到一个单例 Bean 的 map 集合中，key 就是配置的 Bean 的名称。

3. 后置处理器：

   - 每一个 Bean 创建完成，都会使用各种后置处理器进行处理，来增强 Bean 的功能 [AutowiredAnnotationBeanPostProcessor 处理自动注入](Spring Boot 2 注解驱动开发.md#AutowiredAnnotationBeanPostProcessor)，<a href="Spring Boot 2 注解驱动开发.md#AnnotationAwareAspectJAutoProxyCreator">AnnotationAwareAspectJAutoProxyCreator 处理AOP功能</a>

4. 事件驱动模型：

   1. ApplicationListener：事件监听
   2. applicationEventMulticaster：事件派发