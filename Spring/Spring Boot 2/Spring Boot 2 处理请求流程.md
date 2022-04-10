# Spring Boot2 处理请求流程

## 1. DispatchServlet获取用户请求

1. 用户请求被SpringMVC前端控制器**DispatchServlet**类获取。
2. **DispatcherServlet**对请求URL进行解析，得到请求资源标识符（URI）。

3. 判断是否是文件上传

```java
//判断是否是multipart请求，若是，则封装为MultipartHttpServletRequest
processedRequest = checkMultipart(request);
//若是multipart则为true
multipartRequestParsed = (processedRequest != request);
```

## 2. 匹配合适的Handler

  根据该URI，查询**HandlerMappings**获得该Handler配置的所有相关的对象（包含对应请求的控制器方法，所有拦截器集合及拦截器索引(用于执行afterCompletion()方法)），最后以**HandlerExecutionChain**执行链对象的形式返回给**mappedHandler**参数。

<p align="center">mappedHandler中存储的信息</p>

<img src="https://cdn.nlark.com/yuque/0/2021/png/23097143/1635604649378-284c9015-9bdb-4443-92d0-ef7845f59156.png" alt="img" style="zoom: 67%;" />

```java
DispatcherServlet -> doDispatch()
//Return the HandlerExecutionChain for this request.
@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        for (HandlerMapping mapping : this.handlerMappings) {
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
    }
    return null;
}
```

## 3. 根据Handler选择合适的Handler适配器

  DispatcherServlet根据根据获得的Handler，选择一个合适的**HandlerAdapter** 。**(RequestMappingHandlerAdapter)**

![img](https://cdn.nlark.com/yuque/0/2021/png/23097143/1635604417032-e073c2e7-c3b3-48f0-af94-986a3ec460e9.png)

## 4. Handler适配器执行PreHandle()方法

  如果成功获得HandlerAdapter，将执行拦截器的**preHandler()**方法【正向】。若返回为false，则终止执行后续操作。

```java
DispatcherServlet -> doDispatch()
if (!mappedHandler.applyPreHandle(processedRequest, response)) {
    return;
}
```

  顺序执行所有拦截器，当某拦截器返回false时，执行该拦截器之前的所有拦截器的aftercompletion方法，然后退出不执行目标handle()方法

```java
//HandlerExecutionChain
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HandlerInterceptor[] interceptors = getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        //i从0开始正向遍历
        for (int i = 0; i < interceptors.length; i++) {
            HandlerInterceptor interceptor = interceptors[i];
            if (!interceptor.preHandle(request, response, this.handler)) {
                triggerAfterCompletion(request, response, null);
                return false;
            }
            this.interceptorIndex = i;
        }
    }
    return true;
}
```

## 5. Handler适配器执行Handler()方法

  提取Request中的模型数据，填充Handler形参，开始执行**Handler()**方法，处理请求。

```java
//DispatcherServlet -> doDispatch()
// Actually invoke the handler.
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

- 调用**RequestMappingHandlerAdapter**类重写的**handleInternal()**方法。
- 在**handlerInternal()**方法中调用本类的**invokeHandlerMethod()**方法，返回ModelAndView对象。

```java
//RequestMappingHandlerAdapter -> handleInternal()
mav = invokeHandlerMethod(request, response, handlerMethod);
```

### 5.1 添加所有参数解析器和返回值解析器（与handler方法有关）

  进入**invokeHandlerMethod()**方法后，先调用本类的方法获取**databinderfactory**和**modelfactory**

```java
//RequestMappingHandlerAdapter -> invokeHandlerMethod()
WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
```

  然后创建一个**ServletInvocableHandlerMethod**(继承**InvocableHandlerMethod**)类的对象，**invocableMethod**将**参数解析器**和**返回值解析器**，**数据绑定器**及其他信息封装进去

```java
//RequestMappingHandlerAdapter -> invokeHandlerMethod()
ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
/*************************/
if (this.argumentResolvers != null) {
    invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
}
if (this.returnValueHandlers != null) {
    invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
}
invocableMethod.setDataBinderFactory(binderFactory);
```

### 5.2 创建mavContainer

```java
//RequestMappingHandlerAdapter -> invokeHandlerMethod()
ModelAndViewContainer mavContainer = new ModelAndViewContainer();
```

### 5.3 开始执行handle方法

```java
//RequestMappingHandlerAdapter -> invokeHandlerMethod()

//执行handle方法
invocableMethod.invokeAndHandle(webRequest, mavContainer);
if (asyncManager.isConcurrentHandlingStarted()) {
    return null;
}
//返回ModelAndView对象
return getModelAndView(mavContainer, modelFactory, webRequest);
```

  在**invokeAndHandle()**方法中调用父类**InvocableHandlerMethod**的**invokeForRequest()**方法获取执行完handle方法后的返回值对象。

```java
//ServletInvocableHandlerMethod -> invokeAndHandle()

//在该方法中继续调用父类的invokeForRequest方法
Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
```

### 5.4 封装Handler()方法的所有的形参

  在**invokeForRequest()**方法中先调用本类的**getMethodArgumentValues()**方法**封装Handler()方法的所有的形参**，然后执行目标方法。

```java
//InvocableHandlerMethod -> invokeForRequest()
@Nullable
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
    //封装好所有的Handler方法的形参
    Object[] args = this.getMethodArgumentValues(request, mavContainer, providedArgs);
    if (this.logger.isTraceEnabled()) {
        this.logger.trace("Arguments: " + Arrays.toString(args));
    }
	//真正执行目标方法
    return this.doInvoke(args);
}
```

#### 5.4.1 遍历所有的参数解析器找到支持的参数解析器

```java
public boolean supportsParameter(MethodParameter parameter) {
    //不是简单类型，返回！false，于是适用，调用该解析器的resolveArgument方法
    return parameter.hasParameterAnnotation(ModelAttribute.class) || this.annotationNotRequired && !BeanUtils.isSimpleProperty(parameter.getParameterType());
}
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
    MethodParameter[] parameters = this.getMethodParameters();
    if (ObjectUtils.isEmpty(parameters)) {
        return EMPTY_ARGS;
    } else {
        Object[] args = new Object[parameters.length];

        for(int i = 0; i < parameters.length; ++i) {
            MethodParameter parameter = parameters[i];
            parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
            args[i] = findProvidedArgument(parameter, providedArgs);
            if (args[i] == null) {
                //遍历所有的参数解析器找到支持的参数解析器
                if (!this.resolvers.supportsParameter(parameter)) {
                    throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
                }
                try {
                    //解析参数的值
                    args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
                } catch (Exception var10) {
                    if (this.logger.isDebugEnabled()) {
                        String exMsg = var10.getMessage();
                        if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
                            this.logger.debug(formatArgumentError(parameter, exMsg));
                        }
                    }

                    throw var10;
                }
            }
        }

        return args;
    }
}
```

#### 5.4.2 调用对应的参数解析器的resolveArgument()方法解析参数

  转换 -> 绑定

- 例1：**RequestPartMethodArgumentResolver**用于处理文件上传，将Request中的文件信息

- 例2：**modelAttributeMethodProcessor**用于处理bean的自动封装，**当方法的参数是一个自定义bean类型对象时，把他也会自动放到mavContainer中（不会放在model对象中）**

##### 5.4.2.1 创建对应bean的空的对象

  在**resolveArgument()**方法中调用本类的createAttribute方法，利用反射new一个用于封装的对象（调用该实体类的空参构造方法）

```java
attribute = this.createAttribute(name, parameter, binderFactory, webRequest);
protected Object createAttribute(String attributeName, MethodParameter parameter, WebDataBinderFactory binderFactory, NativeWebRequest webRequest) throws Exception {
    MethodParameter nestedParameter = parameter.nestedIfOptional();
    Class<?> clazz = nestedParameter.getNestedParameterType();
    Constructor<?> ctor = BeanUtils.findPrimaryConstructor(clazz);
    if (ctor == null) {
      	//调用空参的构造方法
        Constructor<?>[] ctors = clazz.getConstructors();
        if (ctors.length == 1) {
            ctor = ctors[0];
        } else {
            try {
                ctor = clazz.getDeclaredConstructor();
            } catch (NoSuchMethodException var10) {
                throw new IllegalStateException("No primary or default constructor found for " + clazz, var10);
            }
        }
    }
    Object attribute = this.constructAttribute(ctor, attributeName, parameter, binderFactory, webRequest);
    if (parameter != nestedParameter) {
        attribute = Optional.of(attribute);
    }

    return attribute;
}
```

##### 5.4.2.2 创建WebDataBinder数据绑定器

  在resolveArgument()方法中继续创建**WebDataBinder数据绑定器**，用于将请求参数的值绑定到指定的javaBean的属性中

```java
if (bindingResult == null) {
    
    //利用工厂模式创建数据绑定器
    WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
    if (binder.getTarget() != null) {
        if (!mavContainer.isBindingDisabled(name)) {
            
            //调用bindRequestParameters()方法找到对应的convert转化器并绑定数据
            this.bindRequestParameters(binder, webRequest);
        }
        this.validateIfApplicable(binder, parameter);
        if (binder.getBindingResult().hasErrors() && this.isBindExceptionRequired(binder, parameter)) {
            throw new BindException(binder.getBindingResult());
        }
    }

    if (!parameter.getParameterType().isInstance(attribute)) {
        attribute = binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
    }

    bindingResult = binder.getBindingResult();
}
```

  调用**bindRequestParameters()**时最终调用**ServletRequestDataBinder**类中的bind()方法

```java
public void bind(ServletRequest request) {
    //获取所有参数
    MutablePropertyValues mpvs = new ServletRequestParameterPropertyValues(request);
    MultipartRequest multipartRequest = (MultipartRequest)WebUtils.getNativeRequest(request, MultipartRequest.class);
    if (multipartRequest != null) {
        this.bindMultipart(multipartRequest.getMultiFileMap(), mpvs);
    }
    this.addBindValues(mpvs, request);
    //执行绑定
    this.doBind(mpvs);
}
```

##### 5.4.2.3 查找合适的convert转换器并执行转换

  为了进行绑定先查找合适的转换器用于将请求参数从**原类型转换（默认String）为对应类型的数值**，调用**TypeConverterDelegate**的convertIfNecessary()方法

```java
if (conversionService.canConvert(sourceTypeDesc, typeDescriptor)) {
    try {
        return conversionService.convert(newValue, sourceTypeDesc, typeDescriptor);
    } catch (ConversionFailedException var14) {
        conversionAttemptEx = var14;
    }
}
```

  先调用**GenericConversionService**类的**find()**方法依次遍历找到合适的convert**（可以定制）**

```java
@Nullable
public GenericConverter find(TypeDescriptor sourceType, TypeDescriptor targetType) {
    List<Class<?>> sourceCandidates = this.getClassHierarchy(sourceType.getType());
    List<Class<?>> targetCandidates = this.getClassHierarchy(targetType.getType());
    Iterator var5 = sourceCandidates.iterator();

    while(var5.hasNext()) {
        Class<?> sourceCandidate = (Class)var5.next();
        Iterator var7 = targetCandidates.iterator();

        while(var7.hasNext()) {
            Class<?> targetCandidate = (Class)var7.next();
            ConvertiblePair convertiblePair = new ConvertiblePair(sourceCandidate, targetCandidate);
            GenericConverter converter = this.getRegisteredConverter(sourceType, targetType, convertiblePair);
            if (converter != null) {
                return converter;
            }
        }
    }

    return null;
}
```

##### 5.4.2.4 向bean中的属性注入值

  注入值时**利用反射执行对应的setValue()方法**，会依次将所有从浏览器传入的参数尝试调用对应的set方法

### 5.5 利用反射执行Handler()方法获取返回值

```java
Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
```

### 5.6 调用handleRetureValue()方法处理返回值

  获取返回值对象后，继续在**invokeAndHandle()**方法中处理返回值

```java
try {
    this.returnValueHandlers.handleReturnValue(
        returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
}
```

  调用**HandlerMethodReturnValueHandlerComposite**类的handleRetureValue()方法处理返回值

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType, ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
    //从所有返回值处理器中获取对应的处理器
    HandlerMethodReturnValueHandler handler = this.selectHandler(returnValue, returnType);
    if (handler == null) {
        throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
    } else {
        //调用对应的返回值处理器的handlerRetureValue()方法
        handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
    }
}
```

#### 5.6.1 获取相应的返回值处理器

  先获取对应的处理器

<img src="https://cdn.nlark.com/yuque/0/2020/png/1354552/1605151728659-68c8ce8a-1b2b-4ab0-b86d-c3a875184672.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_23%2Ctext_YXRndWlndS5jb20g5bCa56GF6LC3%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10" alt="img" style="zoom:67%;" />

- 返回值处理器判断是否支持这种类型返回值 supportsReturnType()
- 返回值处理器调用 handleReturnValue()进行处理

#### 5.6.2 调用对应的返回值处理器的handleRetureValue()方法

  调用对应的返回值处理器的**handleReturnValue()**方法。例：控制器方法由**@ResponseBody**或者**@RestController**标识时，由**RequestResponseBodyMethodProcessor**类（继承**AbstractMessageConverterMethodProcessor**类）来处理，返回字符串或没有返回值时，由**ViewNameMethodReturnValueHandler**来处理

#### 5.6.2.1 控制器方法由**@ResponseBody**或者**@RestController**标识时

- 使用**消息转换器**进行写出操作

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
                              ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
    throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
    mavContainer.setRequestHandled(true);
    ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
    ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);
    
	//使用消息转换器进行写出操作
    // Try even with null return value. ResponseBodyAdvice could get involved.
    writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}
```

- 实际进行写出操作时，调用父类**AbstractMessageConverterMethodProcessor**的writeWithMessageConverters()方法

```java
//writeWithMessageCOnverters()方法内
HttpServletRequest request = inputMessage.getServletRequest();

//获得浏览器传入的接受的媒体类型(mediaType)
List<MediaType> acceptableTypes = getAcceptableMediaTypes(request);

//获得可产生的媒体类型（与返回值类型有关）(mediaType)
List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);

if (body != null && producibleTypes.isEmpty()) {
    throw new HttpMessageNotWritableException(
        "No converter found for return value of type: " + valueType);
}
List<MediaType> mediaTypesToUse = new ArrayList<>();

//将浏览器接受的类型和可产生的类型进行匹配，结果存储到mediaTypesToUse集合中
for (MediaType requestedType : acceptableTypes) {
    for (MediaType producibleType : producibleTypes) {
        if (requestedType.isCompatibleWith(producibleType)) {
            mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
        }
    }
}
```

- 查找浏览器可接受的类型时，调用内容协商管理器contentNegotiationManager，底层在HeaderContentNegotiationStrategy的resolveMediaTypes()方法中调用request.getHeaderValues("Accept")，即默认使用基于请求头的策略

```java
private List<MediaType> getAcceptableMediaTypes(HttpServletRequest request)
    throws HttpMediaTypeNotAcceptableException {
    return this.contentNegotiationManager.resolveMediaTypes(new ServletWebRequest(request));
}
```

- 还可开启基于参数策略，但该策略仅支持xml和json，可以修改内容协商策略用来匹配自定义的Converter

```yaml
spring:
    contentnegotiation:
      favor-parameter: true  #开启请求参数内容协商模式
```

​	发请求：http://localhost:8080/test/person?format=json；

​		   http://localhost:8080/test/person?format=xml

- 返回xml格式时需要引入xml相关依赖

```xml
<dependency>
  <groupId>com.fasterxml.jackson.dataformat</groupId>
  <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

![img](https://cdn.nlark.com/yuque/0/2020/png/1354552/1605260623995-8b1f7cec-9713-4f94-9cf1-8dbc496bd245.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_18%2Ctext_YXRndWlndS5jb20g5bCa56GF6LC3%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

- 查找可生产的类型时，遍历所有的messageConverters查找支持的messageConverter转换器（**可以定制**）

```java
for (HttpMessageConverter<?> converter : this.messageConverters) {
    //调用相应的converter.canWrite()方法查询支持写入相应的类型的converter
    if (converter instanceof GenericHttpMessageConverter && targetType != null) {
        if (((GenericHttpMessageConverter<?>) converter).canWrite(targetType, valueClass, null)) {
            result.addAll(converter.getSupportedMediaTypes());
        }
    }
    else if (converter.canWrite(valueClass, null)) {
        result.addAll(converter.getSupportedMediaTypes());
    }
}
```

- 返回对象为Bean时，使用**Json Media Type**

![img](https://cdn.nlark.com/yuque/0/2021/png/23097143/1635756652057-f4bd84cd-0803-4385-afcd-a09503949b28.png)

- 匹配完**可产生的类型**和**浏览器可接受的类型**后，再次循环遍历所有messageConverters查找支持相应类型转换的Converters，然后调用对应转换器的write方法完成写入

```java
genericConverter.write(body, targetType, selectedMediaType, outputMessage);
```

- 例：当返回对象为Json时，最终调用**AbstractJackson2HttpMessageConverter**的WriteInternal()方法，在该方法中完成写入操作

![img](https://cdn.nlark.com/yuque/0/2021/png/23097143/1635757296300-0b2e620e-925c-4ca1-b25e-c38f3c18c50a.png)

#### 5.6.2.2 返回字符串或没有返回值时

  由**ViewNameMethodReturnValueHandler**来处理

- 把ViewName放在mavContainer中

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
                              ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

    if (returnValue instanceof CharSequence) {
        String viewName = returnValue.toString();
        mavContainer.setViewName(viewName);
        
        //判断是否是redirect
        if (isRedirectViewName(viewName)) {
            mavContainer.setRedirectModelScenario(true);
        }
    }
    else if (returnValue != null) {
        // should not happen
        throw new UnsupportedOperationException("Unexpected return type: " +
                                                returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
    }
}
```

## 6 Handler执行完成后返回一个ModelAndView对象

```java
//RequestMappingHandlerAdapter -> handleInternal()
mav = invokeHandlerMethod(request, response, handlerMethod);

//RequestMappingHandlerAdapter -> invokeHandlerMethod()

invocableMethod.invokeAndHandle(webRequest, mavContainer);
return getModelAndView(mavContainer, modelFactory, webRequest);
```

## 7 Handler适配器执行拦截器的postHandle()方法【逆向】

```java
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv)
    throws Exception {

    HandlerInterceptor[] interceptors = getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        for (int i = interceptors.length - 1; i >= 0; i--) {
            HandlerInterceptor interceptor = interceptors[i];
            interceptor.postHandle(request, response, this.handler, mv);
        }
    }
}
```

## 8 有异常时执行异常拦截（可定制）

### 8.1 拦截异常

  执行目标方法，目标方法运行期间有任何异常都会被catch、而且标志当前请求结束；并且用 **dispatchException**

```java
DispatcherServlet -> doDispatch()
catch (Exception ex) {
    dispatchException = ex;
}
catch (Throwable err) {
    // As of 4.3, we're processing Errors thrown from handler methods as well,
    // making them available for @ExceptionHandler methods and other scenarios.
    dispatchException = new NestedServletException("Handler dispatch failed", err);
}
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
```

### 8.2 在处理派发结果时进行异常解析

  处理异常并返回ModelAndView对象。

```java
//DispatcherServlet -> processDispatchResult()
    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException) {
            logger.debug("ModelAndViewDefiningException encountered", exception);
            mv = ((ModelAndViewDefiningException) exception).getModelAndView();
        }
        else {
            Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
            mv = processHandlerException(request, response, handler, exception);
            errorView = (mv != null);
        }
    }
```

#### 8.2.1 遍历所有异常解析器

  默认有2组解析器：**DefaultErrorAttributes**和**HandlerExceptionResolverComposite**(ExceptionHandlerExceptionResolver, ResponseStatusExceptionResolver, DefaultHandlerExceptionResolver)

```java
//DispatcherServlet -> processHandlerException()
if (this.handlerExceptionResolvers != null) {
    for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
        exMv = resolver.resolveException(request, response, handler, ex);
        if (exMv != null) {
            break;
        }
    }
}
```

#### 8.2.2 DefaultErrorAttributes先处理异常

  向request域中添加错误信息，返回null，继续执行后续的异常解析器

```java
public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
    this.storeErrorAttributes(request, ex);
    return null;
}

private void storeErrorAttributes(HttpServletRequest request, Exception ex) {
    request.setAttribute(ERROR_ATTRIBUTE, ex);
}
```

#### 8.2.3 后面没有能解析该异常(例:zero error)的解析器

(**HandlerExceptionResolverComposite**中的解析器)

- processHandlerException()方法最终抛出异常
- 被doDispatch()方法捕获并结束当此请求

##### ==missingServletRequestParameterException==

没有找到标注了`@requestParam`注解的参数时抛出该异常

##### ==NoHandlerFoundException==

> ```java
> WebMvcAutoConfiguration.class -> WebMvcAutoConfigurationAdapter.class
> 
> // 在启动容器时运行
> public void addResourceHandlers(ResourceHandlerRegistry registry) {
>  // 如果 spring.web.resources.add-mappings 设置为 false，则禁用所有静态规则，即无法访问静态资源
>  if (!this.resourceProperties.isAddMappings()) {
>      logger.debug("Default resource handling disabled");
>  } else {
>      Duration cachePeriod = this.resourceProperties.getCache().getPeriod();
>      CacheControl cacheControl = this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl();
>      if (!registry.hasMappingForPattern("/webjars/**")) {
>          this.customizeResourceHandlerRegistration(registry.addResourceHandler(new String[]{"/webjars/**"}).addResourceLocations(new String[]{"classpath:/META-INF/resources/webjars/"}).setCachePeriod(this.getSeconds(cachePeriod)).setCacheControl(cacheControl));
>      }
> 
>      String staticPathPattern = this.mvcProperties.getStaticPathPattern();
>      // 如果 ResourceHandlerRegistry 中没有包含 staticPathPattern(/**)，则自动为 ResourceHandlerRegistry 添加该地址
>      // 该值由 WebMvcProperties 的 staticPathPattern 获得，通过 spring.mvc.static-path-pattern= 设置，默认全匹配，即 /**
>      if (!registry.hasMappingForPattern(staticPathPattern)) {
>          this.customizeResourceHandlerRegistration(registry.addResourceHandler(new String[]{staticPathPattern}).addResourceLocations(WebMvcAutoConfiguration.getResourceLocations(this.resourceProperties.getStaticLocations())).setCachePeriod(this.getSeconds(cachePeriod)).setCacheControl(cacheControl));
>      }
>  }
> }
> ```
>
> 当没有找到匹配 urlpattern 的 handler 时，默认由静态资源处理器 `ResourceHttpRequestHandler` 来匹配(/**)
>
> 所以可通过设置 ==spring.web.resources.add-mappings=false==，或者 修改默认静态资源路径 ==spring.mvc.static-path-pattern=/static== 取消 ResourceHttpRequestHandler 对所有url的匹配
>
> ```java
> DispacherServlet.class -> doDispatch()
> 
> // Determine handler for the current request.
> mappedHandler = getHandler(processedRequest);
> if (mappedHandler == null) {
>  noHandlerFound(processedRequest, response);
>  return;
> }
> ```
>
> 没有找到对应的 handler 时，默认不返回异常，直接发送sendError()，可通过设置 ==spring.mvc.throw-exception-if-no-handler-found=true==，使其产生异常
>
> 出现异常时，最终调用 `processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException)`
>
> 继续调用 `mv = processHandlerException(request, response, handler, exception)`
>
> 遍历所有异常解析器解析异常，被 `HandlerExceptionResolverComposite` 的 `ExceptionHandlerExceptionResolver` 解析（@ControllerAdvise + @ExceptionHandler）
>
> ```java
> DispatcherServlet.class
> 
> protected void noHandlerFound(HttpServletRequest request, HttpServletResponse response) throws Exception {
>  if (pageNotFoundLogger.isWarnEnabled()) {
>      pageNotFoundLogger.warn("No mapping for " + request.getMethod() + " " + getRequestUri(request));
>  }
>  // 如果 spring.mvc.throw-exception-if-no-handler-found 设置为 true，则 throwExceptionIfNoHandlerFound = true
>  if (this.throwExceptionIfNoHandlerFound) {
>      throw new NoHandlerFoundException(request.getMethod(), getRequestUri(request), new ServletServerHttpRequest(request).getHeaders());
>  }
>  else {
>      response.sendError(HttpServletResponse.SC_NOT_FOUND);
>  }
> }
> ```

#### 8.2.4 重新发送error请求

- 底层发送/error请求，会被容器中的**BasicErrorController**处理
- 根据客户端类型不同返回不同的内容：网页 -> 网页内容；postman -> Json数据

- 调用父类AbstractErrorController的resolveErrorView()方法，遍历所有的ErrorViewResolver
- 默认只有一个**DefaultErrorViewResolver**

#### 8.2.5 DefaultErrorViewResolver处理异常

  把响应状态码作为错误页的地址，；例：error/500.html，模板引擎最终响应这个页面。

  若没有对应的错误页，则响应默认白页

## 9 执行processDispatchResult()方法处理派发结果

```java
//DispatcherServlet -> doDispatch()
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
```

## 9.1根据Model和View来渲染视图

  根据返回的ModelAndView（此时会判断是否存在异常：若存在异常，则执行HandlerExceptionResolver进行异常处理）选择一个适合的ViewResolver进行视图解析（**当返回值为json时mv为空，不会经过render渲染**）

```java
//DispatcherServlet -> processDispatchResult()
if (mv != null && !mv.wasCleared()) {
    render(mv, request, response);
    if (errorView) {
        WebUtils.clearErrorRequestAttributes(request);
    }
}
```

### 9.1.1 根据方法的string返回值得到View对象

  定义了页面的渲染逻辑，如决定是response.write()写出或者转发，重定向等。

```xml
//DispatcherServlet -> render()
view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
```

- 遍历所有的视图解析器尝试是否能根据当前字符串返回值得到view对象

```java
//DispatcherServlet -> resolveViewName()
protected View resolveViewName(String viewName, @Nullable Map<String, Object> model,
                               Locale locale, HttpServletRequest request) throws Exception {

    if (this.viewResolvers != null) {
        for (ViewResolver viewResolver : this.viewResolvers) {
            View view = viewResolver.resolveViewName(viewName, locale);
            if (view != null) {
                return view;
            }
        }
    }
    return null;
}
```

- 例：返回"redirect:/main.html"时，由**ContentNegotiationViewResolver**新建一个RedirectView()解析器匹配，返回"forward:/main.html"时，新建一个InternalResourceView()解析器。（有thymeleaf时先创建thymeleafView()，然后在thymeleaf中新建redirectView()或InternalResourceView()）

### 9.1.2 调用父类AbstractView类的render方法

```java
//DispatcherServlet -> render()
view.render(mv.getModelInternal(), request, response);
//AbstractView
/**
	 * Prepares the view given the specified model, merging it with static
	 * attributes and a RequestContext attribute, if necessary.
	 * Delegates to renderMergedOutputModel for the actual rendering.
	 * @see #renderMergedOutputModel
	 */
@Override
public void render(@Nullable Map<String, ?> model, HttpServletRequest request,
                   HttpServletResponse response) throws Exception {

    if (logger.isDebugEnabled()) {
        logger.debug("View " + formatViewName() +
                     ", model " + (model != null ? model : Collections.emptyMap()) +
                     (this.staticAttributes.isEmpty() ? "" : ", static attributes " + this.staticAttributes));
    }
    
    //将model的数据封装
    Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
    prepareResponse(request, response);
    
    //调用对应的View解析器的renderMergedOutputModel方法
    renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
}
```

- 例：转发(forward:/)时使用 **InternalResourceView**

```java
//InternalResourceView
protected void renderMergedOutputModel(
    Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {

    // Expose the model object as request attributes.
    exposeModelAsRequestAttributes(model, request);

    // Determine the path for the request dispatcher.
    String dispatcherPath = prepareForRendering(request, response);

    // If already included or response already committed, perform include, else forward.
    if (useInclude(request, response)) {
        response.setContentType(getContentType());
        if (logger.isDebugEnabled()) {
            logger.debug("Including [" + getUrl() + "]");
        }
        rd.include(request, response);
    }
    else {
        // Note: The forwarded resource is supposed to determine the content type itself.
        if (logger.isDebugEnabled()) {
            logger.debug("Forwarding to [" + getUrl() + "]");
        }
        //request.getRequestDispatcher(path).forward(request, response)
        rd.forward(request, response);
    }
}
```

- 调用父类的exposeModelAsRequestAttributes()方法，**将model中的属性添加到request域中**

```java
//AbstractView
// Expose the model object as request attributes.
exposeModelAsRequestAttributes(model, request);

protected void exposeModelAsRequestAttributes(Map<String, Object> model,
                                              HttpServletRequest request) throws Exception {

    model.forEach((name, value) -> {
        if (value != null) {
            request.setAttribute(name, value);
        }
        else {
            request.removeAttribute(name);
        }
    });
}
```

- 例：重定向(redirecct:/)时使用**RedirectView**

```java
//RedirectView
protected void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request,
                                       HttpServletResponse response) throws IOException {
	//获取目标Url地址
    String targetUrl = createTargetUrl(model, request);
    targetUrl = updateTargetUrl(targetUrl, model, request, response);

    // Save flash attributes
    RequestContextUtils.saveOutputFlashMap(targetUrl, request, response);
	
    // Redirect
    sendRedirect(request, response, targetUrl, this.http10Compatible);
}
//RedirectView -> sendRedirect()
response.sendRedirect(encodedURL);
```

## 10 执行拦截器的afterCompletion()方法【逆向】

- 前面步骤有任何异常都会直接执行afterCompletion()方法
- 执行顺序为**从最后无异常的或最后prehandle()方法返回值为true的拦截器的aftercompletion()方法开始倒叙执行**

```java
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex)
    throws Exception {

    HandlerInterceptor[] interceptors = getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        for (int i = this.interceptorIndex; i >= 0; i--) {
            HandlerInterceptor interceptor = interceptors[i];
            try {
                interceptor.afterCompletion(request, response, this.handler, ex);
            }
            catch (Throwable ex2) {
                logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
            }
        }
    }
}
```

## 11 将渲染结果返回给客户端