## 使用ConfigMap作为配置中心

[参考](https://docs.spring.io/spring-cloud-kubernetes/reference/property-source-config/configmap-propertysource.html)

读取配置规则：

- if you have a *single* entry in the configmap (like in your second example that works), we will not care about anything: we will simply parse that yaml into properties
-  If you have *multiple* entries *and* they end in `yaml/yml/properties`, we will only take it for further parsing if it matches your `${spring.application.name}` or this name with profiles (if it’s not present, by `application.yaml/properties`)
- Apply as a properties file the content of the above name + each active profile

An example should make a lot more sense. Let’s suppose that `spring.application.name=my-app` and that we have a single active profile called `k8s`. For a configuration as below:

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: my-app
data:
  my-app.yaml: |-
    ...
  my-app-k8s.yaml: |-
    ..
  my-app-dev.yaml: |-
   ..
```

These is what we will end-up loading:

- `my-app.yaml` treated as a file
- `my-app-k8s.yaml` treated as a file
- `my-app-dev.yaml` *ignored*, since `dev` is *not* an active profile

### 1.挂载configmap

#### 1.1 读取properties

**pom.xml**

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-kubernetes-client-all</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

**application.yaml**

```yaml
spring:
  application:
    name: cloud-k8s-app
  config:
    import: kubernetes:,configtree:/tmp/
```

**Deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloud-k8s-app
spec:
  selector:
    matchLabels:
      app: cloud-k8s-app
  template:
    metadata:
      labels:
        app: cloud-k8s-app
    spec:
      containers:
      - name: cloud-k8s-app
        image: openjdk:17
        command: ["java", "-jar", "/app/app.jar"]
        volumeMounts:
        - name: app-volume
          mountPath: /app/app.jar
          # 将 configmap 挂载到 /tmp/props 目录下
        - name: configmap-volume
          mountPath: /tmp/props
      volumes:
      - name: app-volume
        hostPath:
          path: /root/app/app.jar
      # 创建 configmap 存储卷
      - name: configmap-volume
        configMap:
          defaultMode: 420
          name: cloud-k8s-app
      # 需要绑定 configmap 等访问权限	
      serviceAccountName: user
```

**configmap.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloud-k8s-app
data:
  application.yaml: |-
    my.config.test: dumy
```

**测试代码**

```java
@RestController
public class TestController {

    @Autowired
    Environment env;

    @Value("${my.config.test}")
    String test;

    @Autowired
    DemoConfig demoConfig;

    @GetMapping("/getEnv")
    public String getEnv() {
        return "config from env: " + env.getProperty("my.config.test")
                + "\nconfig from value: " + test
                + "\nconfig from config: " + demoConfig.getTest();
    }
}

@Configuration
@ConfigurationProperties("my.config")
@Data
public class DemoConfig {

    private String test;
}
```

#### 1.2 热更新properties

热更新需要引入[spring-cloud-kubernetes-configuration-watcher](https://docs.spring.io/spring-cloud-kubernetes/reference/spring-cloud-kubernetes-configuration-watcher.html)，注意修改版本！而后在 `Deployment` 中添加环境变量：

```yaml
env:
# 指定监控的 namespace
- name: SPRING_CLOUD_KUBERNETES_RELOAD_NAMESPACES_0
  value: "default"
# 指令刷新配置的延迟时间
- name: SPRING_CLOUD_KUBERNETES_CONFIGURATION_WATCHER_REFRESHDELAY
  value: "1000"

# 以下为 DEBUG 配置
#- name: LOGGING_LEVEL_ORG_SPRINGFRAMEWORK_CLOUD_KUBERNETES_CONFIGURATION_WATCHER
#  value: DEBUG
#- name: LOGGING_LEVEL_ORG_SPRINGFRAMEWORK_CLOUD_KUBERNETES_CLIENT_CONFIG_RELOAD
#  value: DEBUG
#- name: LOGGING_LEVEL_ORG_SPRINGFRAMEWORK_CLOUD_KUBERNETES_COMMONS_CONFIG_RELOAD
#  value: DEBUG
```

**pom.xml**

引入 `actuator` 的依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

**application.yaml**

添加暴露的端口和开启 `refresh` 和 `restart`

```yaml
spring:
  application:
    name: cloud-k8s-app
  config:
    import: kubernetes:,configtree:/tmp/
management:
  endpoint:
    refresh:
      enabled: true
    restart:
      enabled: true
  endpoints:
    web:
      exposure:
        include:
        - "refresh"
        - "restart"
```

**configmap.yaml**

需要添加两个标签：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloud-k8s-app
  namespace: default
  labels:
    # 必须
    spring.cloud.kubernetes.config: "true"
    # 会减少 apiServer 和 watcher 的压力
    spring.cloud.kubernetes.config.informer.enabled: "true"
data:
  application.yaml: |-
    my.config.test: dumy
```

为服务添加 `Service` 资源

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloud-k8s-app
spec:
  selector:
    matchLabels:
      app: cloud-k8s-app
  template:
    metadata:
      labels:
        app: cloud-k8s-app
    spec:
      containers:
      - name: cloud-k8s-app
        image: openjdk:17
        command: ["java", "-jar", "/app/app.jar"]
        volumeMounts:
        - name: app-volume
          mountPath: /app/app.jar
        - name: configmap-volume
          mountPath: /tmp/props
      volumes:
      - name: app-volume
        hostPath:
          path: /root/app/app.jar
      - name: configmap-volume
        configMap:
          # 为文件增加权限
          defaultMode: 420
          name: cloud-k8s-app
      serviceAccountName: user
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: cloud-k8s-app
  name: cloud-k8s-app
spec:
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  selector:
    app: cloud-k8s-app
  type: ClusterIP
```

> **注意**
>
> 默认情况下 ` @Value("${my.config.test}")` 不会动态更新，需要在类上标注 `@RefreshScope` 开启动态更新：
>
> ```java
> @RestController
> @RefreshScope
> public class TestController {
> 
> @Value("${my.config.test}")
> String test;
> ...
> }
> ```
>
> 当服务名和 ConfigMap 名不同时，需要在 `ConfigMap` 中添加以下 `annotation`：
>
> ```yaml
> spring.cloud.kubernetes.configmap.apps: service_name
> ```

### 2.非挂载方式

**application.yaml**

```yaml
spring:
  application:
    name: cloud-k8s-app
  config:
    # 使用 kubernetes
    import: "kubernetes:"
  # 配置 configMap 名称和命名空间
  cloud:
    kubernetes:
      config:
        enabled: true
        name: ${spring.application.name}
        namespace: default
        # sources:
        # - name:
        #   namespace:
management:
  endpoint:
    refresh:
      enabled: true
    restart:
      enabled: true
  endpoints:
    web:
      exposure:
        include:
          - "refresh"
          - "restart"
```

**Deployment & Service**

不需要 mount ConfigMap

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloud-k8s-app
spec:
  selector:
    matchLabels:
      app: cloud-k8s-app
  template:
    metadata:
      labels:
        app: cloud-k8s-app
    spec:
      containers:
      - name: cloud-k8s-app
        image: openjdk:17
        command: ["java", "-jar", "/app/app.jar"]
        volumeMounts:
        - name: app-volume
          mountPath: /app/app.jar
      volumes:
      - name: app-volume
        hostPath:
          path: /root/app/app.jar
      serviceAccountName: user
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: cloud-k8s-app
  name: cloud-k8s-app
spec:
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  selector:
    app: cloud-k8s-app
  type: ClusterIP

```

其他与[热更新properties](#1.2-热更新properties)一致。

> **简化配置文件**
>
> 暴露端口相关配置可以放在 ConfigMap 中，本地配置文件 `application.yaml` 中只需配置以下选项：
>
> ```yaml
> spring:
>   config:
> 	import: "kubernetes:"
>   application:
>     name: cloud-k8s-app
>   cloud:
>     kubernetes:
>       config:
>         enabled: true
>         name: ${spring.application.name}
>         namespace: default
> ```
>
> **ConfigMap**
>
> ```yaml
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: cloud-k8s-app
>   namespace: default
>   labels:
>     spring.cloud.kubernetes.config: "true"
>     spring.cloud.kubernetes.config.informer.enabled: "true"
> data:
>   cloud-k8s-app.yaml: |-
>     management:
>       endpoint:
>         refresh:
>           enabled: true
>         restart:
>           enabled: true
>       endpoints:
>         web:
>           exposure:
>             include:
>          - "refresh"
>          - "restart"
> ```

## 2.挂载Secret

### 2.1 环境变量方式

**application.yaml**

```yaml
spring:
  config:
    import: "kubernetes:"
  application:
    name: cloud-k8s-app
  cloud:
    kubernetes:
      config:
        enabled: true
        name: ${spring.application.name}
        namespace: default
```

**secrets.yaml**

```yaml
apiVersion: v1
data:
  db.password: dGVtcFBhc3N3b3Jk
  db.username: cm9vdA==
kind: Secret
metadata:
  name: mysql-root-authn
  namespace: default
type: Opaque
```

**Deployment & Service**

```yaml
spec:
  containers:
  - name: cloud-k8s-app
    image: openjdk:17
    command: ["java", "-jar", "/app/app.jar"]
    volumeMounts:
    - name: app-volume
      mountPath: /app/app.jar
    # 添加环境变量
    env:
    - name: db.username
      valueFrom:
        secretKeyRef:
          name: cloud-k8s-app
          key: db.username
  volumes:
  - name: app-volume
    hostPath:
      path: /root/app/app.jar
  serviceAccountName: user
```

该方法不支持动态更新。

### 2.2 非映射方式

由 `springc-cloud-kubernetes` 在启动时自动加载

**application.yaml**

```yaml
spring:
  config:
    import: "kubernetes:"
  application:
    name: cloud-k8s-app
  cloud:
    kubernetes:
      config:
        enabled: true
        name: ${spring.application.name}
        namespace: default
      secrets:
        enabled: true
        namespace: default
        sources:
          - name: mysql-root-authn
        # secret 的该选项默认为 false，表示只能通过环境变量绑定
        enable-api: true
```

**secret.yaml**

```yaml
apiVersion: v1
data:
  db.password: dGVtcFBhc3N3b3Jk
  db.username: ZGVtbw==
kind: Secret
metadata:
  annotations:
    # 发生变更时需要通知的服务名
    spring.cloud.kubernetes.secret.apps: cloud-k8s-app
  labels:
    spring.cloud.kubernetes.secret: "true"
    spring.cloud.kubernetes.secret.informer.enabled: "true"
  name: mysql-root-authn
  namespace: default
type: Opaque
```

**Deployment & Service**

移除 `env`

```yaml
spec:
  containers:
  - name: cloud-k8s-app
    image: openjdk:17
    command: ["java", "-jar", "/app/app.jar"]
    volumeMounts:
    - name: app-volume
      mountPath: /app/app.jar
  volumes:
  - name: app-volume
    hostPath:
      path: /root/app/app.jar
  serviceAccountName: user
```

## 3.监听配置变化

与 NACOS 相同，都是借助 Spring Cloud 默认的实现，当环境发生变化时 `ContextRefresher` 会调用 `refreshEnvironment()` 方法发布 `EnvironmentChangeEvent` 时间，可以创建监听类监听该事件：

```java
@Component
public class MyListener implements ApplicationListener<EnvironmentChangeEvent> {

    private Environment environment;

    public MyListener(Environment environment) {
        this.environment = environment;
    }

    @Override
    public void onApplicationEvent(EnvironmentChangeEvent event) {
        Set<String> keys = event.getKeys();
        keys.forEach(key -> {
            System.out.println(key);
            System.out.println(environment.getProperty(key));
        });
    }
}
```

通过这种方式可以实现**动态更改线程池参数**。

## 4.挂载方式动态加载日志

使用 `log4j2` 时，引入 `spring-boot-starter-log4j2` 后，除了要排除 `spring-boot-starter-web` 中的 `spring-boot-starter-logging`，还要排除 `spring-boot-starter-actuator` 中的 `spring-boot-starter-logging`。

### 4.1 监听器（不推荐）

1. 创建新的 `log4j2.properties` Configmap（或者 `log4j2.xml`）

   ```properties
   appender.console.type = Console
   appender.console.name = console
   appender.console.target = SYSTEM_OUT
   appender.console.layout.type = PatternLayout
   appender.console.layout.pattern = [%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5level] [%t] [%c#%M-%L] %m%n
   rootLogger.level = info
   rootLogger.appenderRefs = console
   rootLogger.appenderRef.console.ref = console
   ```

   ```bash
   $ kubectl create configmap log4j2.properties --from-file=path-to-log4j2/log4j2.propperties                                                  
   ```

2. 挂载至容器，并指定 log4j 文件地址

   ```yaml
   spec:
     containers:
     - name: cloud-k8s-app
       image: openjdk:17
       command: ["java","-Dlog4j.configurationFile=file:/app/config/log4j2.properties",  "-jar", "/app/app.jar"]
       volumeMounts:
       - name: app-volume
         mountPath: /app/app.jar
       - name: log4j2
         mountPath: /app/config/
     volumes:
     - name: app-volume
       hostPath:
         path: /root/app/app.jar
     - name: log4j2
       configMap:
         name: log4j2.properties
   ```

3. 添加一个监听器，用于监听 `log4j.xml` 文件并更新 log4j 配置

   ```java
   @Component
   @Slf4j
   public class MyListener implements ApplicationListener<ApplicationReadyEvent> {
   
       @Override
       public void onApplicationEvent(ApplicationReadyEvent applicationReadyEvent) {
           try {
               // 获取文件系统的 WatchService
               WatchService watchService = FileSystems.getDefault().newWatchService();
   
               // 指定要监听的目录
               Path directory = Paths.get("/app/config/");
   
               // 注册目录到 WatchService，并指定要监听的事件类型
               directory.register(watchService, StandardWatchEventKinds.ENTRY_MODIFY);
   
               // 启动监听线程
               Thread watchThread = new Thread(() -> {
                   try {
                       while (true) {
                           // 获取 WatchKey，它表示一组相关的事件
                           WatchKey key = watchService.take();
   
                           // 处理事件
                           for (WatchEvent<?> event : key.pollEvents()) {
                               // 获取事件的种类
                               WatchEvent.Kind<?> kind = event.kind();
                               // 获取事件关联的路径
                               Path changedPath = (Path) event.context();
                               // 在这里处理文件变化事件
                               log.info("Event type: " + kind + ", File changed: " + changedPath);
                               Configurator.reconfigure();
                           }
                           // 重置 WatchKey，以便下一次监听
                           boolean valid = key.reset();
                           if (!valid) {
                               // 如果 WatchKey 无效，可能是由于目录不再存在，退出监听
                               break;
                           }
                       }
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
               });
               // 启动监听线程
               watchThread.start();
           } catch (Exception e) {
               e.printStackTrace();
           }
       }
   }
   ```

### 4.2 monitorInterval自动检测

When configured from a File, Log4j has the ability to automatically detect changes to the configuration file and reconfigure itself. If the `monitorInterval` attribute is specified on the configuration element and is set to a non-zero value then the file will be checked the next time a log event is evaluated and/or logged and the monitorInterval has elapsed since the last check. The example below shows how to configure the attribute so that the configuration file will be checked for changes only after at least 30 seconds have elapsed. The minimum interval is 5 seconds.

```
<?xml version="1.0" encoding="UTF-8"?><Configuration monitorInterval="30">...</Configuration>
```

1. 创建新的 `log4j2.xml` Configmap

   ```xml
   <?xml version="1.0" encoding="UTF-8" ?>
   
   <Configuration  monitorInterval="5">
   
       <Properties>
           <Property name="pattern" value="[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%-5level] [%t] [%c#%M-%L] %m%n"/>
       </Properties>
   
       <Appenders>
           <Console name="console" target="SYSTEM_OUT">
               <PatternLayout pattern="${pattern}"/>
           </Console>
       </Appenders>
   
       <Loggers>
           <Logger name="com.youyi" level="info" additivity="false">
               <AppenderRef ref="console"/>
           </Logger>
           <Root level="error">
               <AppenderRef ref="console"/>
           </Root>
       </Loggers>
   
   </Configuration>
   ```

2. 挂载至容器，并指定 log4j 文件地址

   ```yaml
   spec:
     selector:
       matchLabels:
         app: cloud-k8s-app
     template:
       metadata:
         labels:
           app: cloud-k8s-app
       spec:
         containers:
         - name: cloud-k8s-app
           image: openjdk:17
           # 指定 log4j 文件
           command: ["java","-Dlog4j.configurationFile=file:/app/config/log4j2.xml",  "-jar", "/app/app.jar"]
           volumeMounts:
           - name: app-volume
             mountPath: /app/app.jar
           # 挂载 log4j configmap
           - name: log4j2
             mountPath: /app/config/
         volumes:
         - name: app-volume
           hostPath:
             path: /root/app/app.jar
         # log4j confgimap 容器卷
         - name: log4j2
           configMap:
             name: log4j2.xml
         serviceAccountName: user
   ```

修改 log4j2.xml Configmap，过段时间（Configmap 同步需要时间，检测到变化也需要时间）发现服务日志已经修改了。

## 注册中心

### 1.OpenFeign

provider 中导入 `spring-cloud-starter-kubernetes-client-all`

```java
@RestController
public class ProviderController  {
    @GetMapping("/getInfo")
    public String getInfo() {
        return "hello world!";
    }
}
```

consumer 中导入 `spring-cloud-starter-kubernetes-client-all` 和 `spring-cloud-starter-openfeign`

启动类上添加 `@EnableFeignClients` 注解

另创建提供者方法接口并在接口上添加 `@FeignClient` 注解

```java
@FeignClient(name = "cloud-k8s-provider")
public interface ProviderService {
    @GetMapping("/getInfo")
    String getInfo();
}
```

```java
@Autowired
ProviderService providerService;

@GetMapping("/getInfo")
public String getInfo() {
    return providerService.getInfo();
}
```

### 2.原生

直接使用 provider 的 service 名称作为目标地址，依靠 k8s 的 DNS 和负载均衡功能来进行访问：

```java
@Autowired
RestTemplate restTemplate;

@GetMapping("/getInfo")
public String getInfo() {
    return restTemplate.getForObject("http://cloud-k8s-provider:8080/getInfo", String.class);
}
```

