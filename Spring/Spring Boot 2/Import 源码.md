# Import源码

## ConfigurationClassPostProcessor

执行`BeanFactory`的后置处理器，在**ConfigurationClassPostProcessor**中解析配置类时，会解析该类是否由`@Import`注解。

```java
public class ConfigurationClassPostProcessor {
    public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
        do {
            StartupStep processConfig = this.applicationStartup.start("spring.context.config-classes.parse");
            // 1.解析配置候选类，注册其 BeanDefinition
            // 2.将配置候选类的 MetaData、resource 等信息存放在缓存 configurationClasses 中
            // 配置候选类是标注了 Confiugration、Component、Serivce、Controller 等注解的类的 BeandefinitionHolder
            // 不包括配置类中的 Bean 注解标注的类
            parser.parse(candidates); // ↓
            parser.validate();
            // 3.从 configurationClasses 中获取所有的配置候选类
            Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
            configClasses.removeAll(alreadyParsed);

            if (this.reader == null) {
                this.reader = new ConfigurationClassBeanDefinitionReader(
                    registry, this.sourceExtractor, this.resourceLoader, this.environment,
                    this.importBeanNameGenerator, parser.getImportRegistry());
            }
            // 4.加载注册其他 BeanDefinitions（Import加载的类的 Definition 在这里被加载）
            this.reader.loadBeanDefinitions(configClasses);
            ...
        }
    }
    while (!candidates.isEmpty());
}

public void parse(Set<BeanDefinitionHolder> configCandidates) {
    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        // 里面调用 ConfigurationClassParser 的 processConfigurationClass()
        parse(bd.getBeanClassName(), holder.getBeanName());
    }
    this.deferredImportSelectorHandler.process();
}
```

```java
class ConfigurationClassParser {
    
    protected void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter) throws IOException {
		...
		do {
            // 解析配置候选类
			sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter); // ↓
		} while (sourceClass != null);
        // 将类的 metadata 和 source 存入缓存中
		this.configurationClasses.put(configClass, configClass);
	}
    
    protected final SourceClass doProcessConfigurationClass() {
        ...      
        // 解析 @Import
		processImports(configClass, sourceClass, getImports(sourceClass), filter, true);
		...
    }
```

## 解析`@Import`

### 获得标注了`@Import`的类

调用`getImports(sourceClass)`

```java
class ConfigurationClassParser {
    
    private Set<SourceClass> getImports(SourceClass sourceClass) throws IOException {
        Set<SourceClass> imports = new LinkedHashSet<>();
        Set<SourceClass> visited = new LinkedHashSet<>();
        collectImports(sourceClass, imports, visited); // ↓
        return imports;
    }
    
    private void collectImports(SourceClass sourceClass, Set<SourceClass> imports, Set<SourceClass> visited)
			throws IOException {
        // 如果是已经访问过的，则 add 方法返回 false，不会再次递归
		if (visited.add(sourceClass)) {
            // 获取类上所有注解
			for (SourceClass annotation : sourceClass.getAnnotations()) {
				String annName = annotation.getMetadata().getClassName();
                // 不是 Import 注解时递归检查该注解上是否有 Import 注解
				if (!annName.equals(Import.class.getName())) {
					collectImports(annotation, imports, visited);
				}
			}
            // 获取所有 Import 注解的 value 值
			imports.addAll(sourceClass.getAnnotationAttributes(Import.class.getName(), "value"));
		}
	}
}
```

### 解析

再调用`processImports()`

```java
class ConfigurationClassParser {
    private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
                                Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,
                                boolean checkForCircularImports) {
		...
        for (SourceClass candidate : importCandidates) {
            // 判断是否是 ImportSelector 的子实现类
            if (candidate.isAssignable(ImportSelector.class)) {
                Class<?> candidateClass = candidate.loadClass();
                // 通过字节码对象反射创建自定义的 ImportSelector 对象
                ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
                                                                               this.environment, this.resourceLoader, this.registry);
                // 获得排除过滤器
                Predicate<String> selectorFilter = selector.getExclusionFilter();
                if (selectorFilter != null) {
                    exclusionFilter = exclusionFilter.or(selectorFilter);
                }
                if (selector instanceof DeferredImportSelector) {
                    this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
                }
                else {
                    // 将标注了 Import 注解的配置类的元数据传入用户实现的 Selector 的 selectImports() 方法
                    String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                    // 根据排除过滤器来过滤需要 import 的类
                    Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, exclusionFilter);
                    // 递归调用自己来解析 importClassNames
                    processImports(configClass, currentSourceClass, importSourceClasses, exclusionFilter, false);
                }
            }
            // 判断是否是 ImportBeanDefinitionRegistrar 的子实现类，是的话存入 importBeanDefinitionRegistrars map 中
            else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                Class<?> candidateClass = candidate.loadClass();
                ImportBeanDefinitionRegistrar registrar =
                    ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
                                                         this.environment, this.resourceLoader, this.registry);
                configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
            }
            else {
                this.importStack.registerImport(
                    currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                // 如果都不是，则按照配置类处理
                // 将其封装为 ConfigurationClass，将导入该类的 configClass 加入 importedBy：Set<ConfigurationClass> 中
                processConfigurationClass(candidate.asConfigClass(configClass), exclusionFilter);
            }
        }
    }
}
```

实际加载`BeanDefinition`是在`ConfigurationClassPostProcessor`的`this.reader.loadBeanDefinitions(configClasses)`中：

```java
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
		TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
		for (ConfigurationClass configClass : configurationModel) {
			loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator); // ↓
		}
	}

    private void loadBeanDefinitionsForConfigurationClass(
			ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {
        // 判断当前配置候选类是否由其他类 import 导入或者是否是某个配置类的 Bean
        // Import 导入的类在这里完成 Beandefinition 加载
		if (configClass.isImported()) {
			registerBeanDefinitionForImportedConfigurationClass(configClass);
		}
	}
```

## ImportSelector

可以根据**字符串数组**（数组元素为类的全类名）来批量的加载指定的 Bean。

### 使用

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

### 源码

```java
class ConfigurationClassParser {
    private Collection<SourceClass> asSourceClasses(String[] classNames, Predicate<String> filter) throws IOException {
        List<SourceClass> annotatedClasses = new ArrayList<>(classNames.length);
        for (String className : classNames) {
            // 遍历所有的 className，排除相应的 className
            annotatedClasses.add(asSourceClass(className, filter));
        }
        return annotatedClasses;
    }
    
    SourceClass asSourceClass(@Nullable String className, Predicate<String> filter) throws IOException {
        // 调用 Predicate 的 test 实现方法来过滤，为 true 时返回封装了 Object 类型的 SourceClass
		if (className == null || filter.test(className)) {
			return this.objectSourceClass;
		}
		if (className.startsWith("java")) {
			// Never use ASM for core java types
			try {
				return new SourceClass(ClassUtils.forName(className, this.resourceLoader.getClassLoader()));
			}
			catch (ClassNotFoundException ex) {
				throw new NestedIOException("Failed to load class [" + className + "]", ex);
			}
		}
		return new SourceClass(this.metadataReaderFactory.getMetadataReader(className));
	}
}
```

