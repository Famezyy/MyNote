# 创建Bean

## 1.【判断是否是FactoryBean】

通过检验是否实现了 FactoryBean 接口判断是否是`FactoryBean`。

## 2.【创建对象】

不是`FactoryBean`则调用父类`AbstractBeanFactory`的`getBean(beanName)`创建对象。

```java
public abstract class AbstractBeanFactory{
    
    public Object getBean(String name) throws BeansException {
        // 最终调用 dogetBean() 方法
        return this.doGetBean(name, (Class)null, (Object[])null, false);
    }
    protected <T> T doGetBean(String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly) throws BeansException {
        String beanName = this.transformedBeanName(name);
        // 2.1 先尝试获取缓存中保存的单实例 Bean，若能获取到则说明之前这个 Bean 被创建过（所有创建过的单实例 Bean 都会被缓存）
        Object sharedInstance = this.getSingleton(beanName);
        Object beanInstance;
        // 判断是否从缓存中获取到了 Bean
        if (sharedInstance != null && args == null) {
            ...
            beanInstance = this.getObjectForBeanInstance(sharedInstance, name, beanName, (RootBeanDefinition)null);
        } else {
            if (this.isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }
// ------------------------------- 2.2 缓存中获取不到，则开始创建 Bean -------------------------------
            BeanFactory parentBeanFactory = this.getParentBeanFactory();
            if (parentBeanFactory != null && !this.containsBeanDefinition(beanName)) {
                String nameToLookup = this.originalBeanName(name);
                if (parentBeanFactory instanceof AbstractBeanFactory) {
                    return ((AbstractBeanFactory)parentBeanFactory).doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
                }

                if (args != null) {
                    return parentBeanFactory.getBean(nameToLookup, args);
                }

                if (requiredType != null) {
                    return parentBeanFactory.getBean(nameToLookup, requiredType);
                }

                return parentBeanFactory.getBean(nameToLookup);
            }
            
			// 2.2.1 标记当前 Bean 已创建
            if (!typeCheckOnly) {
                this.markBeanAsCreated(beanName);
            }
            ...
            try {
                ...
				// 2.2.2 获取 Bean 的定义信息
                RootBeanDefinition mbd = this.getMergedLocalBeanDefinition(beanName);
                ...
                // 2.2.3 获取当前 Bean 依赖的其他 Bean
                String[] dependsOn = mbd.getDependsOn();
                String[] var12;
                // 如果有依赖的 bean，则调用 getBean() 先创建所依赖的 Bean
                if (dependsOn != null) {
                    var12 = dependsOn;
                    int var13 = dependsOn.length;
                    for(int var14 = 0; var14 < var13; ++var14) {
                        String dep = var12[var14];
                        ...
                        this.registerDependentBean(dep, beanName);
                        try {
                            this.getBean(dep);
                        } 
                        ...
                    }
                }
// ------------------------------- 正式开始创建单实例 Bean -------------------------------
                if (mbd.isSingleton()) {
                    // 2.2.4 调用调用父类 DefaultSingletonBeanRegistry 的 getSIngleton()
                    sharedInstance = this.getSingleton(beanName, () -> {
                        try {
                            // 2.2.5 调用子类 AbstractAutowireCapableBeanFactory 的 createBean()
                            return this.createBean(beanName, mbd, args);
                        } catch (BeansException var5) {
                            this.destroySingleton(beanName);
                            throw var5;
                        }
                    });
                    
                    beanInstance = this.getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                } else if (mbd.isPrototype()) {
                    var12 = null;

                    Object prototypeInstance;
                    try {
                        this.beforePrototypeCreation(beanName);
                        prototypeInstance = this.createBean(beanName, mbd, args);
                    } finally {
                        this.afterPrototypeCreation(beanName);
                    }

                    beanInstance = this.getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                } else {
                    ...
                }
            }
            ...
            finally {
                beanCreation.end();
            }
        }
        return this.adaptBeanInstance(name, beanInstance, requiredType);
    }
}
```

### 2.1【尝试获取缓存中保存的单实例Bean】

若能获取到则说明之前这个 Bean 被创建过（所有创建过的单实例 Bean 都会被缓存），则直接返回该 Bean。

这里是利用**三级缓存**（3 个 map）来尝试获取 Bean：

```java
public class DefaultSingletonBeanRegistry {
    public Object getSingleton(String beanName) {
        return this.getSingleton(beanName, true);
    }

    protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null && this.isSingletonCurrentlyInCreation(beanName)) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                synchronized(this.singletonObjects) {
                    singletonObject = this.singletonObjects.get(beanName);
                    if (singletonObject == null) {
                        singletonObject = this.earlySingletonObjects.get(beanName);
                        if (singletonObject == null) {
                            ObjectFactory<?> singletonFactory = (ObjectFactory)this.singletonFactories.get(beanName);
                            if (singletonFactory != null) {
                                singletonObject = singletonFactory.getObject();
                                this.earlySingletonObjects.put(beanName, singletonObject);
                                this.singletonFactories.remove(beanName);
                            }
                        }
                    }
                }
            }
        }

        return singletonObject;
    }
}
```

### 2.2【缓存中获取不到则创建新的Bean】

#### 2.2.1【标记当前Bean已创建】

#### 2.2.2【获取Bean的定义信息】

#### 2.2.3【创建当前Bean依赖的其他Bean】

如果有其他依赖的 Bean，则调用 getBean() 先创建所依赖的 Bean。

#### ==2.2.4【调用父类`DefaultSingletonBeanRegistry`的`getSIngleton()】`==

```java
public class DefaultSingletonBeanRegistry {
    public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        synchronized(this.singletonObjects) {
            // 1.先尝试从缓存中获取
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                ...
                    this.beforeSingletonCreation(beanName);
                // 记录是否创建了新的单例
                boolean newSingleton = false;
                boolean recordSuppressedExceptions = this.suppressedExceptions == null;
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = new LinkedHashSet();
                }

                try {
                    // 2.调用入参 singletonFactory 的 getObject()
                    singletonObject = singletonFactory.getObject();
                    //标记为新的单例
                    newSingleton = true;
                } 
                ...
            } finally {
                ...
                    this.afterSingletonCreation(beanName);
            }
            if (newSingleton) {
                // 3.将创建的 Bean 添加到缓存 singletonObjects 中
                this.addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
}
```

##### 1.【先尝试从缓存中获取Bean】

`singletonObjects`是一个 map。

##### ==2.【调用入参`singletonFactory`接口的`getObject()`】==

这里使用了函数式编程，实际上调用了子类`AbstractAutowireCapableBeanFactory`的`createBean()`（见下）。

##### 3.【将创建的Bean添加到缓存`singletonObjects`中】

> IOC 容器就是这些 Map，许多 Map 中保存了各种 Bean，环境信息等。

##### 4.【返回创建好的Bean】

# createBean()

调用子类`AbstractAutowireCapableBeanFactory`的`createBean()`。

```java
public abstract class AbstractAutowireCapableBeanFactory {
    
    protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
        ...
        RootBeanDefinition mbdToUse = mbd;
        // 1。通过反射得到 bean 的字节码对象
        Class<?> resolvedClass = this.resolveBeanClass(mbd, beanName, new Class[0]);
        ...
        Object beanInstance;
        try {
            // 2.让 BeanPostProcessor 先拦截返回代理对象
            beanInstance = this.resolveBeforeInstantiation(beanName, mbdToUse);
            if (beanInstance != null) {
                // 有代理对象则直接返回
                return beanInstance;
            }
        } catch (Throwable var10) {
            throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName, "BeanPostProcessor before instantiation of bean failed", var10);
        }

        try {
            // 3.如果无代理对象，调用 doCreateBean() 创建 Bean
            beanInstance = this.doCreateBean(beanName, mbdToUse, args);
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Finished creating instance of bean '" + beanName + "'");
            }

            return beanInstance;
        } catch (ImplicitlyAppearedSingletonException | BeanCreationException var7) {
            throw var7;
        } catch (Throwable var8) {
            throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", var8);
        }
    }
}
```

## 1.通过反射【创建bean的字节码对象】



## 2.【BeanPostProcessor先拦截返回代理对象】

```java
public abstract class AbstractAutowireCapableBeanFactory {
    protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
        Object bean = null;
        if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
            if (!mbd.isSynthetic() && this.hasInstantiationAwareBeanPostProcessors()) {
                Class<?> targetType = this.determineTargetType(beanName, mbd);
                if (targetType != null) {
                    // 2.1 执行后置处理器实例化之前的方法
                    bean = this.applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                    // 2.2 如果有返回值，再执行后置处理器初始化之后的方法
                    if (bean != null) {
                        bean = this.applyBeanPostProcessorsAfterInitialization(bean, beanName);
                    }
                }
            }
            mbd.beforeInstantiationResolved = bean != null;
        }

        return bean;
    }
}
```

### 2.1【执行后置处理器实例化之前的方法】

执行<a name="InstantiationAwareBeanPostProcessor">InstantiationAwareBeanPostProcessor</a> 的`postProcessBeforeInstantiation()`方法。

### 2.2【执行后置处理器初始化之后的方法】

如果有返回值，再执行后置处理器初始化之后的方法：`BeanPostProcessor`的`postProcessAfterInitialization(result, beanName)`方法。

### 2.3【有代理对象则直接返回代理对象】

## ==3.【调用doCreateBean()创建Bean】==

如果无代理对象，调用`doCreateBean(beanName, mbdToUse, args)`创建 Bean。

```java
public abstract class AbstractAutowireCapableBeanFactory {
    protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
        BeanWrapper instanceWrapper = null;
        if (mbd.isSingleton()) {
            instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
        }

        if (instanceWrapper == null) {
            // 3.1 创建 Bean 实例
            instanceWrapper = this.createBeanInstance(beanName, mbd, args);
        }

        Object bean = instanceWrapper.getWrappedInstance();
        Class<?> beanType = instanceWrapper.getWrappedClass();
        if (beanType != NullBean.class) {
            mbd.resolvedTargetType = beanType;
        }

        synchronized(mbd.postProcessingLock) {
            if (!mbd.postProcessed) {
                try {
                    // 3.2 调用 MergedBeanDefinitionPostProcessors
                    this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                } catch (Throwable var17) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Post-processing of merged bean definition failed", var17);
                }

                mbd.postProcessed = true;
            }
        }

        boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
        if (earlySingletonExposure) {
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
            }

            this.addSingletonFactory(beanName, () -> {
                return this.getEarlyBeanReference(beanName, mbd, bean);
            });
        }

        Object exposedObject = bean;

        try {
            // 3.3 对 Bean 进行属性赋值
            this.populateBean(beanName, mbd, instanceWrapper);
            // 3.4 Bean 初始化
            exposedObject = this.initializeBean(beanName, exposedObject, mbd);
        } catch (Throwable var18) {
            if (var18 instanceof BeanCreationException && beanName.equals(((BeanCreationException)var18).getBeanName())) {
                throw (BeanCreationException)var18;
            }

            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", var18);
        }

        if (earlySingletonExposure) {
            Object earlySingletonReference = this.getSingleton(beanName, false);
            if (earlySingletonReference != null) {
                if (exposedObject == bean) {
                    exposedObject = earlySingletonReference;
                } else if (!this.allowRawInjectionDespiteWrapping && this.hasDependentBean(beanName)) {
                    String[] dependentBeans = this.getDependentBeans(beanName);
                    Set<String> actualDependentBeans = new LinkedHashSet(dependentBeans.length);
                    String[] var12 = dependentBeans;
                    int var13 = dependentBeans.length;

                    for(int var14 = 0; var14 < var13; ++var14) {
                        String dependentBean = var12[var14];
                        if (!this.removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                            actualDependentBeans.add(dependentBean);
                        }
                    }
                    ...
                }
            }
        }

        try {
            // 3.5 注册 Bean 的销毁方法
            this.registerDisposableBeanIfNecessary(beanName, bean, mbd);
            return exposedObject;
        } catch (BeanDefinitionValidationException var16) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", var16);
        }
    }
}
```

### ==3.1【创建Bean实例】==

通过**反射**利用无参构造器创建 Bean 实例。

```java
public abstract class AbstractAutowireCapableBeanFactory {

    protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
        ...
            return this.instantiateBean(beanName, mbd);
    }

    protected BeanWrapper instantiateBean(String beanName, RootBeanDefinition mbd) {
        try {
            Object beanInstance;
            ...
                // 调用 InstantiationStrategy 的方法实例化对象
                beanInstance = this.getInstantiationStrategy().instantiate(mbd, beanName, this);
            BeanWrapper bw = new BeanWrapperImpl(beanInstance);
            this.initBeanWrapper(bw);
            return bw;

        }
    }
}

public class SimpleInstantiationStrategy implements InstantiationStrategy {
    public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
        // 获取字节码
        Class<?> clazz = bd.getBeanClass();
        ...
            // 获取空参构造器
            constructorToUse = clazz.getDeclaredConstructor();
        ...
            // 最终利用反射创建 bean 实例
            return BeanUtils.instantiateClass(constructorToUse, new Object[0]);
    } 
}
```

### 3.2【调用 <span name="MergedBeanDefinitionPostProcessor">MergedBeanDefinitionPostProcessor</span>】

> 在此处允许`postProcessor`修改 Bean 的定义。

### ==3.3【对Bean进行属性赋值】==

```java
public abstract class AbstractAutowireCapableBeanFactory {
    protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		...
        // 3.3.1 调用 InstantiationAwareBeanPostProcessor 的 postProcessAfterInstantiation()
        Iterator var4 = this.getBeanPostProcessorCache().instantiationAware.iterator();
        while(var4.hasNext()) {
            InstantiationAwareBeanPostProcessor bp = (InstantiationAwareBeanPostProcessor)var4.next();
            if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                return;
            }
        }
		...
        // 3.3.2 调用 InstantiationAwareBeanPostProcessor 的 postProcessProperties()
        for(Iterator var9 = this.getBeanPostProcessorCache().instantiationAware.iterator(); var9.hasNext(); pvs = pvsToUse) {
            InstantiationAwareBeanPostProcessor bp = (InstantiationAwareBeanPostProcessor)var9.next();
            pvsToUse = bp.postProcessProperties((PropertyValues)pvs, bw.getWrappedInstance(), beanName);
        }
        ...
        // 3.3.3 底层利用 setter 方法为属性赋值
        this.applyPropertyValues(beanName, mbd, bw, (PropertyValues)pvs);
    }
}
```

#### 3.3.1【执行后置处理器实例化之后的方法】

#### 3.3.2【执行后置处理器的`ProcessProperties()`】

#### 3.3.3 利用setter方法为【属性赋值】

### ==3.4【Bean初始化】==

```java
public abstract class AbstractAutowireCapableBeanFactory {
    protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
        // 3.4.1 执行 Aware 接口的方法 invokeAwareMethods()
        if (System.getSecurityManager() != null) {
            AccessController.doPrivileged(() -> {
                this.invokeAwareMethods(beanName, bean);
                return null;
            }, this.getAccessControlContext());
        } else {
            this.invokeAwareMethods(beanName, bean);
        }

        Object wrappedBean = bean;
        if (mbd == null || !mbd.isSynthetic()) {
            // 3.4.2 执行后置处理器初始化之前的方法
            wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
        }

        try {
            // 3.4.3 执行初始化方法
            this.invokeInitMethods(beanName, wrappedBean, mbd);
        } catch (Throwable var6) {
            throw new BeanCreationException(mbd != null ? mbd.getResourceDescription() : null, beanName, "Invocation of init method failed", var6);
        }

        if (mbd == null || !mbd.isSynthetic()) {
            // 3.4.4 执行后置处理器初始化之后的方法
            wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }

        return wrappedBean;
    }
}
```

#### 3.4.1【执行Aware接口的方法】

`BeanNameAware`、`BeanClassLoaderAware`、`BeanFactoryAware`。

#### 3.4.2【执行后置处理器初始化之前的方法】

`BeanPostProcessor.postProcessBeforeInitialization()`

#### 3.4.3【执行Bean初始化方法】

- `InitializingBean`接口的实现
- 自定义了初始化方法

`InitializingBean.afterPropertiesSet()`

#### 3.4.4【执行后置处理器初始化之后的方法】

`BeanPostProcessor.postProcessAfterInitialization()`

### 3.5【注册Bean的销毁方法】

创建一个`DisposableBeanDapter`的 Bean 来适用该方法。

