# Import使用

## ImportSelector

可以根据**字符串数组**（数组元素为类的全类名）来批量的加载指定的 Bean。

### 基础使用

```java
// 实现 ImportSelector 接口
public class TestSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[] {"com.youyi.Blue", "com.youyi.Red"};
    }
}
```

```java
// 配置类中使用 @Import 加载 ImportSelector
@Configuration
@Import(TestSelector.class)
public class TestConfiguration {}
```

### 进阶使用

将待加载的类写入配置文件`import.property`

```properties
className=com.youyi.Blue, com.youyi.Red
```

```java
// 实现 ImportSelector 接口
public class TestSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        // 读取配置文件的数据
        ResourceBundle rb = ResourceBundle.getBundle("import");
        String className = rb.getString("className");
        // 转换成数组
        String[] classNameArr = className.split(",");
        return classNameArr;
    }
}
```

**排除过滤器**

```java
// 实现 ImportSelector 接口
public class TestSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        ...
    }
    
    // 将上述方法返回值数组进行过滤，为 true 时排除该字符串
    @Override
    public Predict<String> getExclusionFilter() {
        return s -> s.contains("Blue");
    }
}

```

## ImportBeanDefinitionRegistrar

如果要实现动态 Bean 的装载可以使用`ImportBeanDefinitionRegistrar`，尤其是想**装载动态代理对象**的时候。例如 MyBatis 的启动器就是使用它实现了 Mapper 接口的代理对象装载的。需要注意的是，`ImportBeanDefinitionRegistrar`本身并不会封装成`BeanDefinition`。

### 基础使用

```java
public class SimpleRegistrar implements ImportBeanDefinitionRegistrar {
    /**
     *
     * @param importingClassMetadata：导入类的注释元数据
     * @param registry：当前的 BeanDefinition 注册表
     * @param importBeanNameGenerator：导入 Bean 的 Bean 名称生成器策略
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry, BeanNameGenerator importBeanNameGenerator) {
        ImportBeanDefinitionRegistrar.super.registerBeanDefinitions(importingClassMetadata, registry, importBeanNameGenerator);
    }

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 创建 Blue 的 BeanDefinition 对象
        // GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
        // beanDefinition.setBeanClass(Blue.class);
        AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.rootBeanDefinition(Blue.class).getBeanDefinition();
        //注册该 BeanDefinition，指定 Bean 名称
        registry.registerBeanDefinition("blue", beanDefinition);
    }
}
```

```java
// 配置类中使用 @Import 加载 ImportBeanDefinitionRegistrar
@Configuration
@Import(SimpleRegistrar.class)
public class TestConfiguration {}
```

### 进阶使用

配合<a href="Spring 容器创建过程.md#scan">ClassPathBeanDefinitionScanner</a>实现自定义扫描加载。

编写配置类加载自定义的**注册器**和**配置文件**

```java
@Configuration
@Import(SimpleRegistry.class)
// 通过配置文件加载配置项目
@PropertySource("classpath:/scan.properties")
public class TestConfiguration {
}
```

自定义注册器

```java
public class SimpleRegistry implements ImportBeanDefinitionRegistrar {

    private Environment env;

    @Autowired
    public SimpleRegistry(Environment env) {
        this.env = env;
    }

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 创建 scanner 对象，不使用默认过滤器
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(registry, false);
        // 添加自定义过滤器
        scanner.addIncludeFilter(new TypeFilter() {
            @Override
            public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
                AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
                // 只有标注了 MyComponent 注解的类才会被加载
                return annotationMetadata.hasAnnotation(Objects.requireNonNull(env.getProperty("scanComponent")));
            }
        });
        // 进行扫描
        scanner.scan(env.getProperty("scanPocket"));
    }
}
```

自定义注解

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface MyComponent {
}
```

待加载类

```java
@MyComponent
public class Blue {
}
```

### 究极使用

一些对象的创建过程可能比较复杂，我们可以使用`FactoryBean`来实现对象的创建，可以让 Spring 容器需要创建该对象的时候调用`FactoryBean`来实现创建，尤其是一些动态代理对象的创建。

**模拟Mybatis加载创建mapper**

1. 创建自定义 mapper 注解

   ```java
   @Target(ElementType.TYPE)
   @Retention(RetentionPolicy.RUNTIME)
   public @interface MyMapper {
   }
   ```

2. 创建自定义 mapper 接口

   ```java
   @MyMapper
   public interface SelectMapper {
       void select();
   }
   ```

3. 创建`BeanFactory`来实现 mapper 接口的代理

   ```java
   public class MyMapperFactoryBean implements FactoryBean {
   
       private String className;
   
       public MyMapperFactoryBean(String className) {
           this.className = className;
       }
   
       @Override
       public Object getObject() throws Exception {
           Class<?> interfaceClass = Class.forName(className);
           Object proxyInstance = Proxy.newProxyInstance(this.getClass().getClassLoader(), new Class[]{interfaceClass}, new InvocationHandler() {
               @Override
               public Object invoke(Object o, Method method, Object[] objects) throws Throwable {
                   if (method.getName().equals("select")) {
                       System.out.println("retrieve information from DB");
                   }
                   return o;
               }
           });
           return proxyInstance;
   
       }
   
       @Override
       public Class<?> getObjectType() {
           try {
               return Class.forName(className);
           } catch (ClassNotFoundException e) {
               e.printStackTrace();
               return null;
           }
       }
   }
   ```

4. 创建`scanner`继承`ClassPathBeanDefinitionScanner`重写`isCandidateComponent()`方法来实现加载接口

   ```java
   public class MyMapperScanner extends ClassPathBeanDefinitionScanner {
       public MyMapperScanner(BeanDefinitionRegistry registry) {
           super(registry);
       }
   
       public MyMapperScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters) {
           super(registry, useDefaultFilters);
       }
   
       @Override
       protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
           AnnotationMetadata metadata = beanDefinition.getMetadata();
           return metadata.isInterface();
       }
   }
   ```

5. 创建`Registrar`来扫描加载`BeanDefinition`

   ```java
   public class MyMapperRegistrar implements ImportBeanDefinitionRegistrar {
       @Override
       public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
           // 创建 scanner，不使用默认过滤器(只加载标注了@Component等注解的类)，要重写 isCandidateComponent(AnnotatedBeanDefinition) 方法来允许加载接口的 BeanDefinition
           MyMapperScanner scanner = new MyMapperScanner(registry, false);
           // 设置包含过滤器，判断是否有 MyMapper 注解
           scanner.addIncludeFilter(new TypeFilter() {
               @Override
               public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
                   return metadataReader.getAnnotationMetadata().hasAnnotation("com.example.test.MyMapper");
               }
           });
           // 进行扫描
           Set<BeanDefinition> candidateComponents = scanner.findCandidateComponents("com.example.test");
           // 扫描到 BeanDefinition 后转化成 MyMapperFactoryBean Definition 并注册
           for (BeanDefinition candidateComponent : candidateComponents) {
               AbstractBeanDefinition factoryBeanDefinition = BeanDefinitionBuilder
                   .genericBeanDefinition(MyMapperFactoryBean.class)
                   // 将类型传给 FactoryBean 的构造器
                   .addConstructorArgValue(candidateComponent.getBeanClassName())
                   .getBeanDefinition();
               registry.registerBeanDefinition(Objects.requireNonNull(candidateComponent.getBeanClassName()), factoryBeanDefinition);
           }
       }
   }
   ```

6. 在配置类中导入`Registrar`

   ```java
   @Configuration
   @Import(MyMapperRegistrar.class)
   public class MyMapperConfiguration {
   }
   ```

测试：

获得`SelectMapper`的`Bean`执行`select()`方法可以发现执行了动态代理的方法

```bash
retrieve information from DB
```

