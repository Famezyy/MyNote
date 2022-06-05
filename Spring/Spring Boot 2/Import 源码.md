# Import源码

## ConfigurationClassPostProcessor

执行`BeanFactory`的后置处理器时，通过**ConfigurationClassPostProcessor**加载了所有的`BeanDefinition`，最终执行`ConfigurationClassParser`的`processConfigurationClass()`来解析配置类。

```java
class ConfigurationClassParser {
    
    protected void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter) throws IOException {
		...
		do {
            // 解析配置类
			sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
		} while (sourceClass != null);
		this.configurationClasses.put(configClass, configClass);
	}
    
    protected final SourceClass doProcessConfigurationClass() {
        ...      
        // 解析 @Import
		processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

		// 解析 @ImportResource
		AnnotationAttributes importResource =
				AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
		if (importResource != null) {
			String[] resources = importResource.getStringArray("locations");
			Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
			for (String resource : resources) {
				String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
				configClass.addImportedResource(resolvedResource, readerClass);
			}
		}
		...
    }
```

## 先调用getImports(sourceClass)

解析`@Import`时调用`getImports(sourceClass)`来获得带加载的类：

```java
class ConfigurationClassParser {
    
    private Set<SourceClass> getImports(SourceClass sourceClass) throws IOException {
        Set<SourceClass> imports = new LinkedHashSet<>();
        Set<SourceClass> visited = new LinkedHashSet<>();
        collectImports(sourceClass, imports, visited);
        return imports;
    }
    
    private void collectImports(SourceClass sourceClass, Set<SourceClass> imports, Set<SourceClass> visited)
			throws IOException {
        // 如果是已经访问过的，则 add 方法返回 false，不会再次递归
		if (visited.add(sourceClass)) {
            // 获取类上所有注解
			for (SourceClass annotation : sourceClass.getAnnotations()) {
				String annName = annotation.getMetadata().getClassName();
                // 判断是否是 Import 注解
				if (!annName.equals(Import.class.getName())) {
                    // 递归调用，因为注解中可以嵌套了 import，所以需要递归检查
					collectImports(annotation, imports, visited);
				}
			}
            // 获取所有 Import 注解的 value 值
			imports.addAll(sourceClass.getAnnotationAttributes(Import.class.getName(), "value"));
		}
	}
}
```

## 再调用processImports()

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
                ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
                                                                               this.environment, this.resourceLoader, this.registry);
                Predicate<String> selectorFilter = selector.getExclusionFilter();
                if (selectorFilter != null) {
                    exclusionFilter = exclusionFilter.or(selectorFilter);
                }
                if (selector instanceof DeferredImportSelector) {
                    this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
                }
                else {
                    String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                    Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, exclusionFilter);
                    processImports(configClass, currentSourceClass, importSourceClasses, exclusionFilter, false);
                }
            }
            // 判断是否是 ImportBeanDefinitionRegistrar 的子实现类
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
                processConfigurationClass(candidate.asConfigClass(configClass), exclusionFilter);
            }
        }
    }
}
```

