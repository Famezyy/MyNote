# 第06章_SkyWalking

对于一个大型的微服务架构，通常会遇到下面一些问题：

- 如何串联整个调用链路，定位问题？
- 如何滤清各个微服务之间的依赖关系？
- 如何进行各个微服务接口的性能分析？
- 如何跟踪整个业务流程的调用处理顺序？

SkyWalking 是一个国产开源框架，2017 年加入 Apache。它是分布式系统的应用程序性能监视工具，专为微服务、云原生架构和基于容器架构而设计。它同时是一款 APM（Application Performance Management）工具，包括了分布式追踪、性能指标分析、应用和服务依赖分析等。

其主要功能有以下几个方面：

- 多种监控手段，可以通过语言探针（支持多种语言）和 service mesh 获得监控的数据
- 轻量高效，无需大数据平台和大量服务器资源
- 模块化，UI、存储、集群管理都有多种机制可选
- 支持告警
- 优秀的可视化解决方案

[官网](http://skywalking.apache.org)

[文档](https://skywalking.apache.org/docs/main/v9.7.0/readme/)

## 1.链路追踪框架对比

1. Zipkin 是 Twitter 开源的调用链分析工具，目前基于 SpringCloud Sleuth 得到了广泛的应用，特点是轻量、部署简单
2. Pinpoint 是韩国人开源的基于字节码注入的调用链分析及应用监控分析工具。特点是支持多种插件，UI 功能强大，代码无侵入
3. SkyWalking 是国产开源的基于字节码注入的调用链分析及应用监控分析工具。同样支持多种插件，UI 功能强大，代码无侵入，已加入 Apache
4. CAT 是大众点评开源的基于编码和配置的调用链分析，应用监控分析，日志采集，监控报警等一系列的监控平台工具

| 项目             | Cat                                    | Zipkin             | SkyWalking               |
| ---------------- | -------------------------------------- | ------------------ | ------------------------ |
| 调用链可视化     | 是                                     | 是                 | 是                       |
| 聚合报表         | 非常丰富                               | 少                 | 较丰富                   |
| 服务依赖图       | 简单                                   | 简单               | 丰富                     |
| 埋点方式         | 侵入式                                 | 侵入式             | 非侵入式                 |
| VM 监控指标      | 丰富                                   | NA                 | 简单                     |
| 支持语言         | java、.net                             | 丰富               | java/.net/Node.js/php/go |
| 存储机制         | mysql（报表）、本地文件/HDFS（调用链） | 内存、es、mysql 等 | H2、es                   |
| 社区支持         | 主要国内                               | 国外主流           | Apache 支持              |
| APM              | 是                                     | 否                 | 是                       |
| 是否支持 webflux | 是                                     | 是                 | 是                       |
| 对吞吐量影响     | 大                                     | 中                 | 小                       |

##  2.环境搭建

SkyWalking 主要有以下几个组件：

- **skywalking agent**

  和业务系统绑定在一起，负责收集各种监控数据。

- **skywalking oapservice**

  负责处理监控数据，接收 skywalking agent 的监控数据，并存储在数据库中；接受 skywalking webapp 的前端请求，从数据库查询数据，并返回数据给前端。skywalking oapservice 通常以集群形式存在。

- **sky walking webapp**

  前端界面，用于展示数据。

- **数据库**

  用于存储监控数据，可以使用 mysql、ES（推荐） 等。

### 2.1 服务端搭建

#### 1.JAR启动

1. [官网分别下载](https://skywalking.apache.org/downloads/) APM 和 Agent TAR 文件后解压

   目录结构如下：

   - `webapp`：UI 前端的 jar 包和配置文件
   - `opa-libs`：后台应用的 jar 包，以及依赖的 jar 包，其中 server-starter-*.jar 是启动程序
   - `config`：启动后台应用程序的配置文件
   - `bin`：各种启动脚本
     - `oapService.*`：默认使用的后台程序的启动脚本
     - `opaServiceInit.*`：使用 init 模式启动；在此模式下，OAP 服务器启动以执行初始化工作，然后退出
     - `webappService.*`：UI 前端的启动脚本
     - `startup.*`：组合脚本，会同时启动 oapservice 和 webapp
   - `agent`：
     - `skywalking-agent.jar`：代理服务 jar 包
     - `config`：代理服务启动时使用的配置文件
     - `plugins`：包含多个插件，代理服务启动时会加载该目录下的所有 jar 包
     - `optional-plugins`：包含可选插件，当需要支持某种功能时，如 springCloud Gateway，则需要把该对应的 jar 包拷贝到 plugins 目录下
   - `logs`：存储日志信息

2. 执行 `startup.*` 可执行文件

启动后会启动两个服务，一个是 skywalking-opa-server，它会暴露 11800（收集监控数据） 和 12800（接收前端 UI 请求） 两个端口，可在`config/application.yml`中修改；一个是 skywalking-web-ui，暴露端口默认 8080，可在 `webapp/application.yml` 中修改。

#### 2.Docker启动

**backend startup**

```bash
docker run --name oap --restart always -d \
-p 11800:11800 \
-p 12800:12800 \
apache/skywalking-oap-server:latest
```

**UI startup**

```bash
docker run --name oap-ui --restart always -d \
-p 9090:8080 \
-e SW_OAP_ADDRESS=http://192.168.11.100:12800 \
-e SW_ENABLE_UPDATE_UI_TEMPLATE=true \
apache/skywalking-ui:latest
```

- ` SW_ENABLE_UPDATE_UI_TEMPLATE`：允许修改 UI

只能当感知到相应的服务后才会在 UI 上显示相应的标签。

#### 3.kubernetes启动

[参考](https://skywalking.apache.org/docs/main/v9.7.0/en/setup/backend/backend-k8s/)

### 2.2 接入微服务

#### 1.JAR启动

有以下 2 种实现方式：

- 写一个 shell 脚本，在启动项目的 shell 脚本中，通过`-javaagent`参数配置 SkyWalking Agent。

  ```bash
  #!/bin/bash
  export SW_AGENT_NAME=springboot-skywalking-demo # agent 名称，一般使用 spring.application.name
  export SW_AGENT_COLLECTOR_BACKEND_SERVICES=127.0.0.1:11800 # 配置 Collecttor 地址
  export SW_AGENT_SPAN_LIMIT=2000 # 配置链路的最大 Span 数量，默认 300
  export JAVA_AGENT=-javaagent:/path-to-skywalking/skywalking-agent.jar
  java $JAVA_AGENT -jar springboot-skywalking-demo-0.0.1-SNAPSHOT.jar
  ```

- 直接在命令中声明

  ```bash
  java -javaagent:/path-to-skywalking/skywalking-agent.jar \
  -DSW_AGENT_COLLECTOR_BACKEND_SERVICES=127.0.0.1:11800 \
  -DSW_AGENT_NAME=springboot-skywalking-demo \
  -DSW_AGENT_SPAN_LIMIT=2000 \
  -jar springboot-skywalking-demo-0.0.1-SNAPSHOT.jar
  ```

其中的属性均为 `agent/config/agent.config` 配置中的属性。

#### 2.Docker启动

```bash
FROM apache/skywalking-java-agent:8.5.0-jdk8

# ... build your java application
```

> **注意**
>
> SkyWalking 默认不会显示 SpringCloud gateway 的链路信息，需要将 SpringCLoud Gateway 插件从 `optional-plugins` 中移动到 `plugins` 中。

需要接入多个微服务时，需要为每个微服务添加启动参数。

#### 3.Kubernetes启动

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: agent-as-sidecar
spec:
  restartPolicy: Never

  volumes:
    - name: skywalking-agent
      emptyDir: { }

  initContainers:
    - name: agent-container
      image: apache/skywalking-java-agent:8.7.0-alpine
      volumeMounts:
        - name: skywalking-agent
          mountPath: /agent
      command: [ "/bin/sh" ]
      args: [ "-c", "cp -R /skywalking/agent/agent/" ]

  containers:
    - name: app-container
      image: springio/gs-spring-boot-docker
      volumeMounts:
        - name: skywalking-agent
          mountPath: /skywalking
      env:
        - name: JAVA_TOOL_OPTIONS
          value: "-javaagent:/skywalking/agent/skywalking-agent.jar"
```

## 3.持久化

默认使用 `h2` 内存数据库存储，推荐使用 `ElesticSearch`。

### 3.1 基于Mysql持久化

需要提前创建数据库，表会被 SkyWalking 创建。

1. **JAR 包启动 oap**

   修改 config 目录下的 `application.yaml` 的 `storage.selector` 属性为 `mysql`，并配置响应参数。

2. **Docker 启动 oap**

   ```bash
   docker run --name oap --restart always -d \
   -p 11800:11800 \
   -p 12800:12800 \
   -e SW_STORAGE=mysql \
   -e SW_JDBC_URL=jdbc:mysql://192.168.11.100:3306/sky?rewriteBatchedStatements=true \
   -e SW_DATA_SOURCE_USER=root \
   -e SW_DATA_SOURCE_PASSWORD=root \
   apache/skywalking-oap-server:latest
   ```

> **注意**
>
> 默认 skywalking 没有 mysql 的驱动，需要手动放入 `oap-libs` 目录下：
>
> ```bash
> [root@localhost Downloads]# docker cp mysql-connector-java-8.0.30.jar oap:/skywalking/oap-libs
> Successfully copied 2.52MB to oap:/skywalking/oap-libs
> 
> [root@localhost Downloads]# docker restart oap
> ```

## 4.自定义链路追踪

SkyWalking 默认只追踪 Controller 的调用链路，并且不会记录入参和出参，使用自定义链路追踪可以实现对项目中的任意方法进行链路追踪、参数打印，方便排查问题。

1. **引入依赖**

   ```xml
   <!-- https://mvnrepository.com/artifact/org.apache.skywalking/apm-toolkit-trace -->
   <dependency>
       <groupId>org.apache.skywalking</groupId>
       <artifactId>apm-toolkit-trace</artifactId>
       <version>9.1.0</version>
   </dependency>
   ```

2. 在相应方法上添加 `@Trace` 注解

3. 需要记录返回值和参数时需要添加 `@Tag` 注解，`key` 为界面显示时的名称，通常设置为方法名

   ```java
   @Trace
   @Tag(key="list", value="returnedObj")
   public List<User> list() {
       return UserMapper.list();
   }
   
   @Trace
   @Tags({
           @Tag(key = "getUser", value = "returnedObj"),
           @Tag(key = "param", value = "arg[0]")
   })
   public User getUser(Integer id) {
       return UserMapper.getById(id);
   }
   ```

   - `returnedObj`：表示记录返回值
   - `arg[0]`：表示记录 0 号参数

> **注意**
>
> 记录对象类型时要注意重写 `toString` 方法，否则只会记录内存地址。

## 5.性能分析

可以用来分析某个端点响应时间

- 打开 "profile" 新建任务，选择目标服务和端点，设置相应的参数
- 访问该端点达到指定的"采样数"
- 查看报告，甚至可以分析结果

## 6.集成日志

[参考：logback](https://skywalking.apache.org/docs/skywalking-java/v9.1.0/en/setup/service-agent/java-agent/application-toolkit-logback-1.x/)

添加依赖

```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-logback-1.x</artifactId>
    <version>${skywalking.version}</version>
</dependency>
```

### 6.1 Print trace ID in your logs

- set `%tid` in `Pattern` section of logback.xml

  ```xml
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
          <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout">
              <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%tid] [%thread] %-5level %logger{36} -%msg%n</Pattern>
          </layout>
      </encoder>
  </appender>
  ```

- with the MDC, set `%X{tid}` in `Pattern` section of logback.xml

  ```xml
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
          <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.mdc.TraceIdMDCPatternLogbackLayout">
              <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{tid}] [%thread] %-5level %logger{36} -%msg%n</Pattern>
          </layout>
      </encoder>
  </appender>
  ```

- Support logback AsyncAppender(MDC also support), No additional configuration is required. Refer to the demo of logback.xml below. For details: [Logback AsyncAppender](https://logback.qos.ch/manual/appenders.html#AsyncAppender)

  ```xml
  <configuration scan="true" scanPeriod=" 5 seconds">
      <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
          <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
              <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.mdc.TraceIdMDCPatternLogbackLayout">
                  <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{tid}] [%thread] %-5level %logger{36} -%msg%n</Pattern>
              </layout>
          </encoder>
      </appender>
  
      <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
          <discardingThreshold>0</discardingThreshold>
          <queueSize>1024</queueSize>
          <neverBlock>true</neverBlock>
          <appender-ref ref="STDOUT"/>
      </appender>
  
      <root level="INFO">
          <appender-ref ref="ASYNC"/>
      </root>
  </configuration>
  ```

- When you use `-javaagent` to active the SkyWalking tracer, logback will output **traceId**, if it existed. If the tracer is inactive, the output will be `TID: N/A`.

### 6.2 gRPC reporter

Your only need to replace pattern `%tid` or `%X{tid]}` with `%sw_ctx` or `%X{sw_ctx}`

The gRPC reporter could forward the collected logs to SkyWalking OAP server, or [SkyWalking Satellite sidecar](https://github.com/apache/skywalking-satellite). Trace id, segment id, and span id will attach to logs automatically. There is no need to modify existing layouts.

- Add `GRPCLogClientAppender` in logback.xml

  ```xml
  <appender name="grpc-log" class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.log.GRPCLogClientAppender">
      <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
          <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.mdc.TraceIdMDCPatternLogbackLayout">
              <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{tid}] [%thread] %-5level %logger{36} -%msg%n</Pattern>
          </layout>
      </encoder>
  </appender>
  ```

- Add config of the plugin or use default

  ```properties
  log.max_message_size=${SW_GRPC_LOG_MAX_MESSAGE_SIZE:10485760}
  ```

## 7.告警功能

[文档](https://skywalking.apache.org/docs/main/v9.7.0/en/setup/backend/backend-alarm/)

The alerting core is driven by a collection of rules defined in `config/alarm-settings.yml`, you have to enable it if you want to use this feature. There are three parts to alerting rule definitions.

1. [alerting rules](https://skywalking.apache.org/docs/main/v9.7.0/en/setup/backend/backend-alarm/#rules)

   They define how metrics alerting should be triggered and what conditions should be considered.

2. [hooks](https://skywalking.apache.org/docs/main/v9.7.0/en/setup/backend/backend-alarm/#hooks) / webhooks

   The list of hooks, which should be called after an alerting is triggered.

### 7.1 默认规则

For convenience’s sake, we have provided a default `alarm-setting.yml` in our release. It includes the following rules:

1. Service average response time over 1s in the last 3 minutes
2. Service success rate lower than 80% in the last 2 minutes
3. Percentile of service response time over 1s in the last 3 minutes
4. Service Instance average response time over 1s in the last 2 minutes, and the instance name matches the regex
5. Endpoint average response time over 1s in the last 2 minutes
6. Database access average response time over 1s in the last 2 minutes
7. Endpoint relation average response time over 1s in the last 2 minutes

### 7.2 Webhook

The Webhook requires the peer to be a web container. The alarm message will be sent through HTTP post by `application/json` content type. The JSON format is based on `List<org.apache.skywalking.oap.server.core.alarm.AlarmMessage>` with the following key information:

- **scopeId**, **scope**

  All scopes are defined in `org.apache.skywalking.oap.server.core.source.DefaultScopeDefine`.

- **name**

  Target scope entity name. Please follow the [entity name definitions](https://skywalking.apache.org/docs/main/v9.7.0/en/setup/backend/backend-alarm/#entity-name).

- **id0**

  The ID of the scope entity that matches with the name. When using the relation scope, it is the source entity ID.

- **id1**

  When using the relation scope, it is the destination entity ID. Otherwise, it is empty

- **ruleName**

  The rule name configured in `alarm-settings.yml`.

- **alarmMessage**

  The alarm text message.

- **startTime**

  The alarm time measured in milliseconds, which occurs between the current time and the midnight of January 1, 1970 UTC

- **tags**

  The tags configured in `alarm-settings.yml`.

See the following example:

```json
[{
  "scopeId": 1, 
  "scope": "SERVICE",
  "name": "serviceA", 
  "id0": "12",  
  "id1": "",  
    "ruleName": "service_resp_time_rule",
  "alarmMessage": "alarmMessage xxxx",
  "startTime": 1560524171000,
    "tags": [{
        "key": "level",
        "value": "WARNING"
     }]
}, {
  "scopeId": 1,
  "scope": "SERVICE",
  "name": "serviceB",
  "id0": "23",
  "id1": "",
    "ruleName": "service_resp_time_rule",
  "alarmMessage": "alarmMessage yyy",
  "startTime": 1560524171000,
    "tags": [{
        "key": "level",
        "value": "CRITICAL"
    }]
}]
```

a sample configuration is as below:

```yaml 
webhook:
  - http://192.168.11.97:9999/getInfo
```

## 8.高可用

[文档](https://skywalking.apache.org/docs/main/v9.7.0/en/setup/backend/backend-cluster/)