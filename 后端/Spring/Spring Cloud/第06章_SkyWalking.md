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

官网：http://skywalking.apache.org

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

1. [官网下载](https://skywalking.apache.org/downloads/) tar 文件后解压

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

2. 执行`startup.*`可执行文件

启动后会启动两个服务，一个是 skywalking-opa-server，它会暴露 11800（收集监控数据） 和 12800（接收前端 UI 请求） 两个端口，可在`config/application.yml`中修改；一个是 skywalking-web-ui，暴露端口默认 8080，可在`webapp/webapp.yml`中修改。

### 2.2 接入微服务

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

其中的属性均为`agent/config/agent.config`配置中的属性。

> **注意**
>
> SkyWalking 默认不会显示 SpringCloud gateway 的链路信息，需要将 SpringCLoud Gateway 插件从`optional-plugins`中移动到`plugins`中。

需要接入多个微服务时，需要为每个微服务添加启动参数。

## 3.持久化

## 4.日志

## 5.告警