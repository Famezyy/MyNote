# 第01章_SpringCloud入门简介

## 1.微服务简介

### 1.1 系统架构演变

系统架构大体经历了下面几个过程：

单体应用架构 - 垂直应用架构 - 分布式架构 - SOA 架构 - 微服务架构，还有悄然兴起的 Service Mesh。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202301262347479.png" alt="image-20230126234741229" style="zoom:67%;" />

#### 1.单体应用架构

互联网早期，一般的网站应用流量较小，只需一个应用，将所有功能代码都部署在一起就可以，这样可以减少开发、部署和维护的成本。比如说一个电商系统，里面会包含很多用户管理，商品管理，订单管理，物流管理等等很多模块，我们会把它们做成一个 web 项目，然后部署到一台 tomcat 服务器上。

**优点:**

- 项目架构简单，小型项目的话，开发成本低
- 项目部署在一个节点上，维护方便

**缺点：**

- 全部功能集成在一个工程中，对于大型项目来讲不易开发和维护
- 项目模块之间紧密耦合，单点容错率低
- 无法针对不同模块进行针对性优化和水平扩展

#### 2.垂直应用架构

随着访问量的逐渐增大，单一应用只能依靠增加节点来应对，但是这时候会发现并不是所有的模块都会有比较大的访问量.

还是以上面的电商为例子，用户访问量的增加可能影响的只是用户和订单模块，但是对消息模块，的影响就比较小。那么此时我们希望只多增加几个订单模块，而不增加消息模块. 此时单体应用就做不到了，垂直应用就应运而生了。

所谓的垂直应用架构，就是将原来的一个应用拆成互不相干的几个应用以提升效率。比如我们可以将上面电商的单体应用拆分成：

- 电商系统（用户管理 商品管理 订单管理）
- 后台系统（用户管理 订单管理 客户管理）
- CMS 系统（广告管理 营销管理）

这样拆分完毕之后，一旦用户访问量变大，只需要增加电商系统的节点就可以了，而无需增加后台和 CMS 的点。

**优点：**

- 系统拆分实现了流量分担，解决了并发问题，而且可以针对不同模块进行优化和水扩展一个系统的问题不会影响到其他系统，提高容错率

**缺点：**

- 系统之间相互独立，无法进行相互调用

- 系统之间相互独立，会有重复的开发任务

#### 3.分布式架构

当垂直应用越来越多，重复的业务代码就会越来越多。这时候我们就思考可不可以将重复的代码抽取出来做成统一的业务层作为独立的服务，然后由前端控制层调用不同的业务层服务呢?

这就产生了新的分布式系统架构。它将把工程拆分成表现层和服务层两个部分，服务层中包含业务逻辑。表现层只需要处理和页面的交互，业务逻辑都是调用服务层的服务来实现。

**优点：**

- 抽取公共的功能为服务层，提高代码复用性

**缺点：**

- 系统间耦合度变高，调用关系错综复杂，难以维护

#### 4.SOA架构

在分布式架构下，当服务越来越多，容量的评估，小服务资源的浪费等问题逐渐显现，此时需增加一个调度中心对集群进行实时管理。此时，用于资源调度和治理中心（SOA Service Oriented Architecture）是关键。

**优点：**

- 使用治理中心（ESB\dubbo）解决了服务间调用关系的自动调节

**缺点：**

- 服务间会有依赖关系，一旦某个环节出错会影响较大（服务雪崩）

- 服务关系复杂，运维、测试部署困难

#### 5.微服务架构

微服务架构在某种程度上是 SOA 架构继续发展的下一步，它更加强调服务的"彻底拆分"。

**微服务架构与SOA架构的不同**

微服务架构比 SOA 架构粒度会更加精细，让专业的人去做专业的事情，目的提高效率，每个服务于服务之间互不影响，微服务架构中，每个服务必须独立部署，微服务架构更加轻巧，轻量级。

SOA 架构中可能数据库存储会发生共享，微服务强调独每个服务都是单独数据库，保证每个服务于服务之间互不影响。项目体现特征微服务架构比 SOA 架构更加适合与互联网公司敏捷开发、快速迭代版本，因为粒度非常精细。

**优点：**

- 服务原子化拆分，独立打包、部署和升级，保证每个微服务清晰的任务划分，利于扩展微服务之间采用 Restful 等轻量级 http 协议相互调用

**缺点：**

- 分布式系统开发的技术成本高（容错、分布式事务等）

- 复杂性更高，各个微服务进行分布式独立部署，当进行模块调用的时候，分布式将会变得更加麻烦

### 1.2 微服务架构

作者：Martin Fowler

英文：https://martinfowler.com/articles/microservices.html

中文：http://blog.cuicc.com/blog/2015/07/22/microservices

他说微服务其实是一种架构风格，我们在开发一个应用的时候这个应用应该是由一组小型服务组成，每个小型服务都运行在自己的进程内，小服务之间通过 HTTP 的方式进行互联互通。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202301270004998.png" alt="image-20230127000422978"  />

#### 1.微服务架构常见问题

一旦采用微服务系统架构，就势必会遇到这样几个问题：

- 管理 - 服务治理、注册中心【服务注册 发现 剔除】：nacos

- 通讯 - restful、rpc、dubbo、feign：`httpclient("url",参数)`in Java；`restTemplate("url",参数)`in springboot；`feign` in springcloud

- 网关 - gateway
- 容错 - sentinel

- 链路追踪 - skywalking


#### 2.常见微服务架构

- dubbo：zookeeper +dubbo + SpringMVC/SpringBoot
  - 通信方式：rpc
  - 注册中心：zookeeper/redis
  - 配置中心：diamond

- SpringCloud：全家桶+轻松嵌入第三方组件（Netflix）（Netflix 已不再更新）
  - 通信方式：http restful
  - 注册中心：eruka/consul
  - 配置中心：config
  - 断路器：hystrix
  - 网关：zuul
  - 分布式追踪系统：sleuth + zipkin

- SpringCloud Alibaba

## 2.pring Cloud简介

SpringCloud 是分布式微服务架构的一站式解决方案，是多种微服务架构落地技术的集合体，俗称微服务全家桶。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/20200603155055794-fc07cd451761e5e06eb2c996682d5ef7-eec8bd.png" alt="img" style="zoom: 33%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/2020060315584195-ea7d2e5f5061ee36b08eb3cac7b6be3d-cc347c.png" alt="在这里插入图片描述" style="zoom: 33%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/20200603160047209-8a41543f45157695aba72b6e87d9519b-12c8c1.png" alt="在这里插入图片描述" style="zoom:33%;" />

**spring cloud 与 spring boot 版本的对应关系**

官方对应关系：https://spring.io/projects/spring-cloud#overview

|                        Release Train                         |             Boot Version              |
| :----------------------------------------------------------: | :-----------------------------------: |
| [2022.0.x](https://github.com/spring-cloud/spring-cloud-release/wiki/Spring-Cloud-2022.0-Release-Notes) aka Kilburn |                 3.0.x                 |
| [2021.0.x](https://github.com/spring-cloud/spring-cloud-release/wiki/Spring-Cloud-2021.0-Release-Notes) aka Jubilee | 2.6.x, 2.7.x (Starting with 2021.0.3) |
| [2020.0.x](https://github.com/spring-cloud/spring-cloud-release/wiki/Spring-Cloud-2020.0-Release-Notes) aka Ilford | 2.4.x, 2.5.x (Starting with 2020.0.3) |
| [Hoxton](https://github.com/spring-cloud/spring-cloud-release/wiki/Spring-Cloud-Hoxton-Release-Notes) |   2.2.x, 2.3.x (Starting with SR5)    |
| [Greenwich](https://github.com/spring-projects/spring-cloud/wiki/Spring-Cloud-Greenwich-Release-Notes) |                 2.1.x                 |
| [Finchley](https://github.com/spring-projects/spring-cloud/wiki/Spring-Cloud-Finchley-Release-Notes) |                 2.0.x                 |
| [Edgware](https://github.com/spring-projects/spring-cloud/wiki/Spring-Cloud-Edgware-Release-Notes) |                 1.5.x                 |
| [Dalston](https://github.com/spring-projects/spring-cloud/wiki/Spring-Cloud-Dalston-Release-Notes) |                 1.5.x                 |

详细对应关系：https://start.spring.io/actuator/info

```json
"spring-cloud": {
    "Hoxton.SR12": "Spring Boot >=2.2.0.RELEASE and <2.4.0.M1",
    "2020.0.6": "Spring Boot >=2.4.0.M1 and <2.6.0-M1",
    "2021.0.0-M1": "Spring Boot >=2.6.0-M1 and <2.6.0-M3",
    "2021.0.0-M3": "Spring Boot >=2.6.0-M3 and <2.6.0-RC1",
    "2021.0.0-RC1": "Spring Boot >=2.6.0-RC1 and <2.6.1",
    "2021.0.5": "Spring Boot >=2.6.1 and <3.0.0-M1",
    "2022.0.0-M1": "Spring Boot >=3.0.0-M1 and <3.0.0-M2",
    "2022.0.0-M2": "Spring Boot >=3.0.0-M2 and <3.0.0-M3",
    "2022.0.0-M3": "Spring Boot >=3.0.0-M3 and <3.0.0-M4",
    "2022.0.0-M4": "Spring Boot >=3.0.0-M4 and <3.0.0-M5",
    "2022.0.0-M5": "Spring Boot >=3.0.0-M5 and <3.0.0-RC1",
    "2022.0.0-RC1": "Spring Boot >=3.0.0-RC1 and <3.0.0-RC2",
    "2022.0.0-RC2": "Spring Boot >=3.0.0-RC2 and <3.0.0",
    "2022.0.0": "Spring Boot >=3.0.0 and <3.1.0-M1"
},
```

每个 SpringCloud 版本有官方建议的 SpringBoot 版本

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230109014639392-297e18d88777a0f60f1a731ad2a06a3c-34fff3.png" alt="image-20230109014639392" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230109014656233-33454c01ff4767331a483fd1f3aa73af-f99a45.png" alt="image-20230109014656233" style="zoom:67%;" />

## 3.Spring Cloud Alibaba

### 3.1 简介

Spring Cloud Alibaba 致力于提供微服务开发的一站式解决方案。此项目包含开发微服务架构的必需组件，方便开发者通过 Spring Cloud 编程模型轻松使用这些组件来开发微服务架构。

依托 Spring Cloud Alibaba，您只需要添加一些注解和少量配置，就可以将 Spring Cloud 应用接入阿里分布式应用解决方案，通过阿里中间件来迅速搭建分布式应用系统。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202301270017733.png" alt="image-20230127001759715" style="zoom:67%;" />

不同版本的服务使用情况

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202301270019547.png" alt="image-20230127001953529" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202301270020054.png" alt="image-20230127002028037" style="zoom:67%;" />

**版本依赖关系**

https://github.com/alibaba/spring-cloud-alibaba/wiki/

| Spring Cloud Alibaba Version | Spring Cloud Version  | Spring Boot Version |
| ---------------------------- | --------------------- | ------------------- |
| 2021.0.4.0*                  | Spring Cloud 2021.0.4 | 2.6.11              |
| 2021.0.1.0                   | Spring Cloud 2021.0.1 | 2.6.3               |
| 2021.1                       | Spring Cloud 2020.0.1 | 2.4.2               |

### 3.2 案例：升级旧微服务架构

#### 1.旧架构使用`ip`访问

- **创建父项目**

  POM 文件

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  	<modelVersion>4.0.0</modelVersion>
  	<parent>
  		<groupId>org.springframework.boot</groupId>
  		<artifactId>spring-boot-starter-parent</artifactId>
  		<version>2.6.11</version>
  		<relativePath/> <!-- lookup parent from repository -->
  	</parent>
  	<groupId>com.youyi.zhao</groupId>
  	<artifactId>springcloudalibaba</artifactId>
  	<version>0.0.1-SNAPSHOT</version>
  	<name>spring-cloud-test</name>
  	<description>hello world project project for Spring Cloud Alibaba</description>
  	<packaging>pom</packaging>
  	<properties>
  		<java.version>1.8</java.version>
  	</properties>
  	<dependencies>
  		<dependency>
  			<groupId>org.springframework.boot</groupId>
  			<artifactId>spring-boot-starter-web</artifactId>
  		</dependency>
  
  		<dependency>
  			<groupId>org.springframework.boot</groupId>
  			<artifactId>spring-boot-starter-test</artifactId>
  			<scope>test</scope>
  		</dependency>
  	</dependencies>
  
  	<build>
  		<plugins>
  			<plugin>
  				<groupId>org.springframework.boot</groupId>
  				<artifactId>spring-boot-maven-plugin</artifactId>
  			</plugin>
  		</plugins>
  	</build>
  
  </project>
  ```

- **创建 order 服务**

  POM 文件

  ```xml
  <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
  	  <groupId>com.youyi.zhao</groupId>
  	  <artifactId>springcloudalibaba</artifactId>
  	  <version>0.0.1-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>order</artifactId>
    <name>order</name>
    <description>order service</description>
  </project>
  ```

  Controller

  ```java
  @RestController
  @RequestMapping("/order")
  public class OrderController {
  
      @Autowired
      RestTemplate restTemplate;
  
      @GetMapping("/add")
      public String add() {
  	String msg = restTemplate.getForObject("http://localhost:8081/stock/reduct", String.class);
  	return msg;
      }
  }
  ```

- **创建 stock 服务**

  POM 文件

  ```xml
  <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
      <parent>
  	  <groupId>com.youyi.zhao</groupId>
  	  <artifactId>springcloudalibaba</artifactId>
  	  <version>0.0.1-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>stock</artifactId>
    <name>stock</name>
    <description>stock service</description>
  </project>
  ```

  Controller

  ```java
  @RestController
  @RequestMapping("/stock")
  public class StockController {
  
      @GetMapping("/reduct")
      public String reduct() {
  	return "库存减成功";
      }
  
  }
  ```

访问`http://localhost:8080/order/add`，结果如下：

```bash
库存减成功
```

这种架构耦合性太高，例如 order 服务要维护一个 sotck 服务的地址。

#### 2.使用SpringCloudAlibaba

- **确定版本**

  - Spring Boot：2.6.11

  - Spring Cloud：2021.0.4

  - Spring Cloud Alibaba：2021.0.4.0

- **修改父项目 POM 文件**

  在`dependencyManagement`中添加`spring-cloud-alibaba-dependencies`

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  	<modelVersion>4.0.0</modelVersion>
  	<parent>
  		<groupId>org.springframework.boot</groupId>
  		<artifactId>spring-boot-starter-parent</artifactId>
  		<version>2.6.11</version>
  		<relativePath/> <!-- lookup parent from repository -->
  	</parent>
  	<groupId>com.youyi.zhao</groupId>
  	<artifactId>springcloudalibaba</artifactId>
  	<version>0.0.1-SNAPSHOT</version>
  	<name>spring-cloud-test</name>
  	<description>hello world project project for Spring Cloud Alibaba</description>
  	<packaging>pom</packaging>
  	<properties>
  		<java.version>1.8</java.version>
  		<spring-cloud.version>2021.0.4</spring-cloud.version>
  		<spring-cloud-alibaba.version>2021.0.4.0</spring-cloud-alibaba.version>
  	</properties>
  	
  	<dependencies>
  		<dependency>
  			<groupId>org.springframework.boot</groupId>
  			<artifactId>spring-boot-starter-web</artifactId>
  		</dependency>
  
  		<dependency>
  			<groupId>org.springframework.boot</groupId>
  			<artifactId>spring-boot-starter-test</artifactId>
  			<scope>test</scope>
  		</dependency>
  	</dependencies>
  	
  	<dependencyManagement>
  		<dependencies>
  			<dependency>
  				<groupId>com.alibaba.cloud</groupId>
  				<artifactId>spring-cloud-alibaba-dependencies</artifactId>
  				<version>${spring-cloud-alibaba.version}</version>
  				<type>pom</type>
  				<scope>import</scope>
  			</dependency>
  			<dependency>
                  <groupId>org.springframework.cloud</groupId>
                  <artifactId>spring-cloud-dependencies</artifactId>
                  <version>${spring-cloud.version}</version>
                  <type>pom</type>
                  <scope>import</scope>
              </dependency>
  		</dependencies>
  	</dependencyManagement>
  
  	<build>
  		<plugins>
  			<plugin>
  				<groupId>org.springframework.boot</groupId>
  				<artifactId>spring-boot-maven-plugin</artifactId>
  			</plugin>
  		</plugins>
  	</build>
  
  </project>
  ```
  
  > **提示**
  >
  > 也可使用官方脚手架：`https://start.aliyun.com`

接下来先学习下 Nacos 注册中心。 

### 3.3 注册中心

#### 1.注册中心简介

##### 1.1 演变过程

注册中心用来管理所有微服务、解决微服务之间调用关系错综复杂、难以维护的问题。

1. 将远程调用地址写在代码中，缺点：不易于更新维护

2. 手动维护一份服务地址的注册表，远程调用时去注册表中找相应的地址，缺点：不易于服务的水平扩展

3. 利用 nginx 维护服务地址的注册表，从而实现水平扩展时的负载均衡，缺点：微服务数量较多时配置维护困难

4. 利用注册中心动态维护注册表

   - 各个服务启动时注册服务到注册中心
   - 每次调用服务时从注册中心获取服务列表进行调用 

   缺点：无法监控服务是否宕机。

5. 引入心跳机制，定时监测服务是否运行

##### 1.2 主流的注册中心

|                 |           Nacos            | Eureka（停止维护） |      Consul       |  CoreDNS   | Zookeeper  |
| :-------------: | :------------------------: | :----------------: | :---------------: | :--------: | :--------: |
|   一致性协议    |           CP/AP            |         AP         |        CP         |     -      |     CP     |
|    健康检查     | TCP/HTTP/MYSQL/Client Beat |    Client Beat     | TCP/HTTP/gRPC/Cmd |     -      | Keep Alive |
|  负载均衡策略   |   权重/metadata/Selector   |       Ribbon       |       Fabio       | RoundRobin |     -      |
|    雪崩保护     |             Y              |         Y          |         N         |     N      |     N      |
|  自动注销实例   |             Y              |         Y          |         Y         |     N      |     Y      |
|    访问协议     |          HTTP/DNS          |        HTTP        |     HTTP/DNS      |    DNS     |    TCP     |
|    监听支持     |             Y              |         Y          |         Y         |     N      |     Y      |
|   多数据中心    |             Y              |         Y          |         Y         |     N      |     N      |
| 跨注册中心同步  |             Y              |         N          |         Y         |     N      |     N      |
| SpringCloud集成 |             Y              |         Y          |         Y         |     N      |     Y      |
|    Dubbo集成    |             Y              |         N          |         Y         |     N      |     Y      |
|     K8S集成     |             Y              |         N          |         Y         |     Y      |     N      |

##### 1.3 Nacos Discovery

Nacos 是一种集注册中心 + 配置中心 + 服务管理的平台。

关键特征包括：

- 服务发现和服务健康监测
- 动态配置服务
- 动态 DNS 服务
- 服务及其元数据管理

Nacos Discovery Starter 可以将服务自动注册到 Nacos 服务端并且能够动态感知和刷新某个服务实例的服务列表。除此之外，Nacos Discovery Starter 也将服务实例自身的一些元数据信息，例如 host、port、健康检查URL、主页等注册到 Nacos。

参考：https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-discovery

- 服务注册

  Nacos Client 会通过发送 REST 请求的方式向 Nacos Server 注册自己的服务，提供自身的元数据，比如 ip 地址、端口等信息。 Nacos Server 接收到注册请求后，就会把这些元数据信息存储在一个双层的内存 Map 中。

- 服务心跳

  在服务注册后，Nacos Client 会维护一个定时心跳来持续通知 Nacos Server，说明服务一直处于可用状态，防止被剔除，默认 5s 发送一次心跳。

- 服务同步

  Nacos Server 集群之间会互相同步服务实例，用来保证服务信息的一致性。

- 服务发现

  服务消费者（Nacos Client）在调用服务提供者的服务时，会发送一个 REST 请求给 Nacos Server，获取上面注册的服务清单，并且缓存在 Nacos Client 本地，同时会在 Nacos Client 本地开启一个定时任务定时拉取服务端最新的注册表信息更新到本地缓存。

- 服务健康检查

  Nacos Server 会开启一个定时任务用来检查注册服务实例的健康情况，对于超过 15s 没有收到客户端心跳的实例会将它的 healthy 属性置为 false（客户端服务发现时不会发现），如果某个实例超过 30 秒没有收到心跳，直接剔除该实例（被剔除的实例如果恢复发送心跳则会重新注册）。

####  2.Nacos Server部署

| Spring Cloud Alibaba Version | Sentinel Version | Nacos Version | RocketMQ Version | Dubbo Version | Seata Version |
| ---------------------------- | ---------------- | ------------- | ---------------- | ------------- | ------------- |
| 2.2.10-RC1                   | 1.8.6            | 2.2.0         | 4.9.4            | ~             | 1.6.1         |
| 2022.0.0.0-RC1               | 1.8.6            | 2.2.1-RC      | 4.9.4            | ~             | 1.6.1         |
| 2.2.9.RELEASE                | 1.8.5            | 2.1.0         | 4.9.4            | ~             | 1.5.2         |
| 2021.0.4.0                   | 1.8.5            | 2.0.4         | 4.9.4            | ~             | 1.5.2         |

根据版本依赖关系，这里使用`2.0.4`版本。

##### 2.1 安装

> **注意**
>
> 首先要安装 Java 环境。

参考：https://nacos.io/zh-cn/docs/quick-start.html

你可以通过源码和发行包两种方式来获取 Nacos。

- 从 Github 上下载源码方式

  ```bash
  git clone https://github.com/alibaba/nacos.git
  cd nacos/
  mvn -Prelease-nacos -Dmaven.test.skip=true clean install -U  
  ls -al distribution/target/
  
  # change the $version to your actual path
  cd distribution/target/nacos-server-$version/nacos/bin
  ```

- 下载编译后压缩包方式

  从[最新稳定版本](https://github.com/alibaba/nacos/releases)下载`nacos-server-$version.tar.gz`包。

  ```bash
  unzip nacos-server-$version.zip
  # or tar -xvf nacos-server-$version.tar.gz
  cd nacos/bin
  ```

##### 2.2 运行

> **注意**
>
> Nacos 的运行需要以至少 2C4g60g*3 的机器配置下运行。

- Linux/Unix/Mac

  启动命令（`standalone`代表着单机模式运行，非集群模式）：

  ```bash
  sh startup.sh -m standalone
  ```

  如果使用的是 ubuntu 系统，或者运行脚本报错提示[[符号找不到，可尝试如下运行：

  ``` bash
  bash startup.sh -m standalone
  ```

- Windows

  启动命令：

  ```bash
  startup.cmd -m standalone
  ```

> **创建执行脚本**
>
> 在`/etc/init.d`下创建`nacos`文件
>
> ```bash
> #! /bin/bash
> #chkconfig:2345 80 90
> #description:nacos
> #processname:nacos
> 
> NACOS=/srv/nacos
> case $1 in
>   start) sh $NACOS/bin/startup.sh -m standalone;;
>   stop) sh $NACOS/bin/shutdown.sh;;
>   *) echo "require start|stop";;
> esac
> ```
>
> 把脚本注册为 Service
>
> ```bash
> chkconfig --add nacos
> chkconfig --list
> ```
>
> 增加权限
>
> ```bash
> chmod +x /etc/init.d/nacos
> ```
>
> 启动
>
> ```bash
> service nacos start
> ```

##### 2.3 登陆

等服务器启动成功后，查看输出日志，访问指定地址，用户名和密码默认是`nacos`。

> **注意**
>
> 需要打开防火墙指定端口或关闭防火墙。

##### 2.4 关闭

- Linux/Unix/Mac：`sh shutdown.sh`
- Wubdiws：`shutdown.cmd`

#### 3.Nacos Client部署

改造 order 和 stock 两个微服务。

- 配置服务提供者 stock

  添加`spring-cloud-starter-alibaba-nacos-discovery`依赖

  ```xml
  <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  	<parent>
  	  	<groupId>com.youyi.zhao</groupId>
  	  	<artifactId>springcloudalibaba</artifactId>
  	  	<version>0.0.1-SNAPSHOT</version>
    	</parent>
    	<modelVersion>4.0.0</modelVersion>
    	<artifactId>stock</artifactId>
    	<name>stock</name>
    	<description>stock service</description>
    
    	<dependencies>
  	  	<dependency>
  	        <groupId>com.alibaba.cloud</groupId>
  	        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        	</dependency>
    	</dependencies>
  </project>
  ```

  在`Application.yaml`中配置 Nacos server 地址

  ```yaml
  server:
    port: 8081
  spring:
    application:
      name: stock-service # 服务名称
    cloud:
      nacos:
        server-addr: 192.168.11.100:8848 # 注册中心地址
        discovery:
          username: nacos
          password: nacos
          namespace: public # 命名空间，用来隔离环境
  ```

  通过 Spring Cloud 原生注解`@EnableDiscoveryClient`开启服务注册发现功能

  ```java
  @SpringBootApplication
  @EnableDiscoveryClient
  public class StockApplication {
  ```

  启动服务后即可在 Nacos 的服务列表看到注册的服务

- 配置服务消费者 

  添加 nacos 和 loadbalance 的依赖
  
  ```xml
  <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
     <parent>
  	   <groupId>com.youyi.zhao</groupId>
  	   <artifactId>springcloudalibaba</artifactId>
  	   <version>0.0.1-SNAPSHOT</version>
     </parent>
     <modelVersion>4.0.0</modelVersion>
     <artifactId>order</artifactId>
     <name>order</name>
     <description>order service</description>
    
    	<dependencies>
  		<dependency>
  			<groupId>org.springframework.cloud</groupId>
  			<artifactId>spring-cloud-starter-loadbalancer</artifactId>
  	  	</dependency>
  	  	<dependency>
  	    	<groupId>com.alibaba.cloud</groupId>
  	      	<artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        	</dependency>
    	</dependencies>
  </project>
  ```
  
  在`Application.yaml`中配置 Nacos server 地址
  
  ```yaml
  server:
    port: 8080
  spring:
    application:
      name: order-service # 服务名称
    cloud:
      nacos:
        server-addr: 192.168.11.100:8848 # 注册中心地址
        discovery:
          username: nacos
          password: nacos
          namespace: public # 命名空间，用来隔离环境
  ```
  
  通过 Spring Cloud 原生注解`@EnableDiscoveryClient`开启服务注册发现功能，通过`@LoadBalanced`开启负载均衡功能
  
  ```java
  @SpringBootApplication
  @EnableDiscoveryClient
  public class OrderApplication {
  
      public static void main(String[] args) {
  	SpringApplication.run(OrderApplication.class, args);
      }
  
      @Bean
      @LoadBalanced
      RestTemplate restTemplate(RestTemplateBuilder builder) {
  	return builder.build();
      }
  }
  ```
  
  将调用的 IP 地址换成服务名称
  
  ```java
  @RestController
  @RequestMapping("/order")
  public class OrderController {
  
      @Autowired
      RestTemplate restTemplate;
  
      @GetMapping("/add")
      public String add() {
  	String msg = restTemplate.getForObject("http://stock-service/stock/reduct", String.class);
  	return msg;
      }
  }
  ```

此时访问 order 服务就能调用到 stock 服务。同时，如果启动多台 stock 服务，则默认会使用**轮询的负载均衡策略**。

#### 4.Nacos的配置界面

- 新建命名空间，用来区分不同的生产环境

- 创建空服务，相当于先把服务名创建好，等待注册

- 服务详情

  - 分组：比命名空间更细粒度的分类，设置服务的分组

  - 保护阈值（0～1）

    对于永久实例（配置文件中显式声明`spring.cloud.nacos.discovery.ephemeral=false`），当服务宕机后会被标记为不健康状态而不会从服务列表删除。此时就存在健康服务和不健康服务。

    如果设置了保护阈值，则当“健康实例数/总实例数 < 保护阈值”的时候，nacos 会把该服务所有的实例信息（健康的和不健康的）都提供给消费者，一部分的消费者可能访问不到，但是也比造成雪崩好，保证了整个系统的可用性。

    一般不设置，而是通过 sentinel 来控制服务的熔断。

    > **注意**
    >
    > Nacos 不允许将一个临时实例修改为非临时实例（相同 IP）。
    >
    > 解决方法：
    >
    > - 停掉 Nacos
    > - 删除 Nacos 的`\data\protocol\raft`下的内容
    > - 重启 Nacos

  - 元数据：key-value 键值对，可配合修改源码来提供过滤功能

  - 权重（1-100）：配合负载均衡器使用，设置的越大，调用的频次越高

- 订阅者列表：查询某个服务的被消费记录

#### 5.Nacos的配置项

更多关于 spring-cloud-starter-alibaba-nacos-discovery 的 starter 配置项：https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-discovery

#### 6.Docker下集群模式部署

这里为了简便，在`192.168.11.100`上的 Docker 中部署三个 Nacos 服务

参考：https://nacos.io/zh-cn/docs/quick-start-docker.html

- 创建 MySQL 容器并初始化数据库

  创建容器参考【Docker部署】，创建表：[sql语句源文件](https://github.com/alibaba/nacos/blob/master/distribution/conf/mysql-schema.sql)

- 创建 Nacos 容器

  ```bash
  docker network create mynet
  ```

  Nacos1

  ```bash
  docker run -d \
  -e PREFER_HOST_MODE=ip \
  -e MODE=cluster \
  -e NACOS_SERVERS="177.18.20.11:8848 177.18.20.12:8848" \
  -e SPRING_DATASOURCE_PLATFORM=mysql \
  -e MYSQL_SERVICE_HOST=192.168.11.100 \
  -e MYSQL_SERVICE_PORT=3306 \
  -e MYSQL_SERVICE_DB_NAME=nacos \
  -e MYSQL_SERVICE_USER=root \
  -e MYSQL_SERVICE_PASSWORD=123456 \
  --privileged=true \
  -v /youyi/nacos/nacos01/logs:/home/nacos/logs \
  --name nacos01 \
  --net mynet --ip 177.18.20.10 \
  --restart=always \
  nacos/nacos-server:v2.0.4
  ```

  Nacos2

  ```bash
  docker run -d \
  -e PREFER_HOST_MODE=ip \
  -e MODE=cluster \
  -e NACOS_SERVERS="177.18.20.10:8848 177.18.20.12:8848" \
  -e SPRING_DATASOURCE_PLATFORM=mysql \
  -e MYSQL_SERVICE_HOST=192.168.11.100 \
  -e MYSQL_SERVICE_PORT=3306 \
  -e MYSQL_SERVICE_DB_NAME=nacos \
  -e MYSQL_SERVICE_USER=root \
  -e MYSQL_SERVICE_PASSWORD=123456 \
  --privileged=true \
  -v /youyi/nacos/nacos02/logs:/home/nacos/logs \
  --name nacos02 \
  --net mynet --ip 177.18.20.11 \
  --restart=always \
  nacos/nacos-server:v2.0.4
  ```

  Nacos3

  ```bash
  docker run -d \
  -e PREFER_HOST_MODE=ip \
  -e MODE=cluster \
  -e NACOS_SERVERS="177.18.20.10:8848 177.18.20.11:8848" \
  -e SPRING_DATASOURCE_PLATFORM=mysql \
  -e MYSQL_SERVICE_HOST=192.168.11.100 \
  -e MYSQL_SERVICE_PORT=3306 \
  -e MYSQL_SERVICE_DB_NAME=nacos \
  -e MYSQL_SERVICE_USER=root \
  -e MYSQL_SERVICE_PASSWORD=123456 \
  --privileged=true \
  -v /youyi/nacos/nacos03/logs:/home/nacos/logs \
  --name nacos03 \
  --net mynet --ip 177.18.20.12 \
  --restart=always \
  nacos/nacos-server:v2.0.4
  ```

- 在`/youyi/nginx/conf.d`下创建`default.conf`配置文件

  ```bash
  upstream nacoscluster-8848 {
      server nacos01:8848;
      server nacos02:8848;
      server nacos03:8848;
  }
  
  
  server {
          listen 80;
          server_name 192.168.11.100;
  
          location / {
                  root /usr/share/nginx/html;
                  index index.html index.htm;
          }
  
          error_page 500 502 503 504 /50x.html;
  
          location = /50x.html {
                  root /usr/share/nginx/html;
          }
  
  }
  
  server {
          listen 8848;
          server_name 192.168.11.100;
  
          location /nacos {
                  proxy_pass http://nacoscluster-8848/nacos;
          }
  }
  ```

- 启动容器，将`nginx.conf`取出

  ```bash
  docker run -d \
  --name nginx \
  --net mynet \
  -v /youyi/nginx/conf.d:/etc/nginx/conf.d \
  -v /youyi/nginx/html:/etc/nginx/html \
  -v /youyi/nginx/log:/var/log/nginx \
  -p 8848:8848 \
  -p 9848:9848 \
  -p 9555:9555 \
  nginx
  ```

  ```bash
  docker cp nginx:/etc/nginx/nginx.conf /youyi/nginx/conf/nginx.conf
  ```

- 打开`/youyi/nginx/conf/nginx.conf`，添加三层负载均衡

  > **注意**
  >
  > Nacos2.0 版本相比 1.X 新增了 gRPC 的通信方式，因此需要增加 2 个端口。新增端口是在配置的主端口基础上进行一定偏移量自动生成。
  >
  > | 端口 | 与主端口的偏移量 | 描述                                                         |
  > | :--- | :--------------- | :----------------------------------------------------------- |
  > | 9848 | 1000             | 客户端 gRPC 请求服务端端口，用于客户端向服务端发起连接和请求 |
  > | 9849 | 1001             | 服务端 gRPC 请求服务端端口，用于服务间同步等                 |

  ```bash
  http{
  ...
  }
  
  # 在 http 同级下添加三层负载均衡
  stream {
     	upstream nacoscluster-9848 {
      	server nacos01:9848;
      	server nacos02:9848;
      	server nacos03:9848;
  	}
      	server {
  		listen 9848;
  		proxy_pass nacoscluster-9848;
  	}
      
      upstream nacoscluster-9849 {
      	server nacos01:9848;
      	server nacos02:9848;
      	server nacos03:9848;
      }
      	server {
  		listen 9849;
  		proxy_pass nacoscluster-9849;
  	}
  }
  ```

- 重新创建容器

  ```bash
  docker run -d \
  --name nginx \
  --net mynet \
  -v /youyi/nginx/conf.d:/etc/nginx/conf.d \
  -v /youyi/nginx/html:/etc/nginx/html \
  -v /youyi/nginx/log:/var/log/nginx \
  -v /youyi/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \
  -p 8848:8848 \
  -p 9848:9848 \
  -p 9849:9849 \
  nginx
  ```

将 SpringCloud 配置文件中的`server-addr`修改成 Nginx 的地址。

```yaml
server-addr: http://192.168.11.100:8848
```

### 3.4 负载均衡

目前主流的负载方案：

- 服务端负载均衡：在消费者和服务提供方中间使用独立的代理方式进行负载，分为硬件实现（比如 F5）和软件实现（比如 Nginx）
- 客户端负载均衡：客户端根据请求做负载均衡

**常见的负载均衡算法**

- 随机
- 轮询
- 加权轮询
- 地址哈希
- 最小链接数

#### 1.Ribbon（过时）

> **注意**
>
> Spring Cloud 2020 版本以后，默认移除了对 Netflix 的依赖，其中就包括 Ribbon，官方默认推荐使用 Spring Cloud Loadbalancer 正式替换 Ribbon，并成为了 Spring Cloud 负载均衡器的唯一实现。

Ribbon 是一种基于 Netflix Ribbon 实现的客户端负载均衡。通过 Load Balance 获取到服务提供的所有机器实例并在客户端进行维护，Ribbon 会基于某种规则（轮询、随机）去调用这些服务。Ribbon 也可以实现自己的负载均衡算法。

##### 1.1 常用接口

- `IRule`

  这是所有负载均衡策略的父接口，核心方法是`choose()`，用来选择一个服务实例。

- `AbstractLoadBalancerRule`

  一个抽象类，定义了一个`ILoadalancer`负载均衡器，用来辅助均衡策略选择合适的服务端实例。

  - `RandomRule`：随机选择一个服务实例
  - `RoundRobinRule`：线性轮询负载均衡策略
  - `RetryRule`：在轮询的基础上加上重试功能（根据超时时间判断）
  - `weightedResponseTimeRule`：服务的平均响应时间越多则权重越大
  - `BestAvailableRule`：过滤掉失效的服务实例，找出并发请求最小的服务实例
  - `ZoneAvoidanceRule`（默认）：根据服务器所在的区域的性能和可用性过滤出符合条件的服务器，然后采用轮询策略

##### 1.2 代码示例

- 通过配置类实现随机策略

  ```java
  @Configuration
  public class RandomRuleConfig {
      @Bean
      public IRule iRule() {
          return new RandomRule();
      }
  }
  ```

  > **注意**
  >
  > 配置类不能放到`@ComponentScan`能够扫描到的地方，或者不能加`@Configuration`，否则会被所有的`RibbonClient`共享。

  在配置类上加上`@RibbonClient`注解

  ```java
  @SpringBootApplication
  @EnableDiscoveryClient
  @RibbonClients(value = {
      @RibbonCLient(name = "stock-service", configuration = RandomRuleConfig.class)
  })
  public class OrderApplication {
  ```

- 通过配置文件实现权重策略

  按照设置的实例权重进行分配（可以在 Nacos 管理界面设置权重或者配置文件中设置）

  ```yaml
  stock-service:
  	ribbon:
  		NFLoadBalancerRuleClassName: com.alibaba.cloud.nacos.ribbon.NacosRule
  ```

##### 1.3 自定义负载均衡

实现`AbstractLoadBalancerRule`接口，重写`choose()`方法。

```java
public class CustomRule extends AbstractLoadBalancerRule {
    
    @Override
    public Server choose(Object key) {
        
        ILoadBalancer loadBalancer = this.getLoadBalancer();
        // 获得服务实例
        List<Server> reachableServers = loadBalancer.getReachableServers();
        // 随机选取一个服务实例
        int random = ThreadLocalRandom.current().nextInt(reachableServers.size());
        Server server = reachableSerers.get(random);
        return server;
    }
}
```

接下来通过配置类或者配置文件的方式注册自定义负载均衡策略。

##### 1.4 开启饥饿加载

负载均衡器默认是懒加载的，只有在第一次使用到时才会加载，可通过以下设置改成饥饿加载：

```yaml
ribbon:
	eager-load:
		# 开启饥饿加载
		enabled: true
		# 指定需要饥饿加载的服务名称
		clients: stock-service
```

#### 2.LoadBalancer

Spring Cloud 2020 版本后默认的负载均衡器，也是一种客户端负载均衡实现，官方提供了两种负载均衡客户端：

- `RestTemplate`

  Spring 提供的用于访问 Rest 服务的客户端，默认依赖 JDK 的 HTTP 连接工具。

- `WebClient`

  Spring WebFlux 5.0 开始提供的一个非阻塞的基于响应式编程的 HTTP 连接工具。

**添加依赖**

在服务消费端添加`loadbalancer`的依赖，需要在父 POM 项目中引入`spring-cloud`的依赖。

> **注意**
>
> 如果使用的 spring-cloud 版本中默认继承了 Ribbon，则需要`exclusion`排除 Ribbon 的依赖，并在配置文件中添加`spring.cloud.loadbalancer.ribbon.enabled = false`。

父 POM

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>${spring-cloud-alibaba.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

子 POM

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
</dependencies>
```

##### 2.1 修改负载均衡策略

3.1 版本中只提供了三种默认策略：随机、轮询、权重，可以自定义。4.1 版本中添加了更多的负载均衡策略，并支持配置文件更改。

- 随机均衡策略

  ```java
  public class CustomLoadBalancerConfiguration {
  
      @Bean
      ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(Environment environment,
              LoadBalancerClientFactory loadBalancerClientFactory) {
          String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
          return new RandomLoadBalancer(loadBalancerClientFactory
                  .getLazyProvider(name, ServiceInstanceListSupplier.class),
                  name);
      }
  }
  ```

  > **注意**
  >
  > 上面的类不能加`@Configuration`注解，或者必须放到包扫面路径外，否则会被所有实例共享，下面的例子同理。

  配置类

  ```java
  @Configuration
  @LoadBalancerClients(defaultConfiguration = CustomLoadBalancerConfiguration.class)
  @EnableDiscoveryClient
  public class MyConfiguration {
  
      @Bean
      @LoadBalanced
      RestTemplate restTemplate(RestTemplateBuilder builder) {
  	return builder.build();
      }
  }
  ```

- 权重均衡策略

  ```java
  public class CustomLoadBalancerConfiguration {
  
      @Bean
      ReactorLoadBalancer<ServiceInstance> nacosLoadBalancer(Environment environment,
  	    LoadBalancerClientFactory loadBalancerClientFactory, NacosDiscoveryProperties nacosDiscoveryProperties) {
  	String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
  	return new NacosLoadBalancer(loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
  		name, nacosDiscoveryProperties);
      }
  }
  ```

  ```java
  @Configuration
  @LoadBalancerClients({
  	@LoadBalancerClient(value = "stock-service", configuration = CustomLoadBalancerConfiguration.class) })
  @EnableDiscoveryClient
  public class MyConfiguration {
  
      @Bean
      @LoadBalanced
      RestTemplate restTemplate(RestTemplateBuilder builder) {
  	return builder.build();
      }
  }
  ```

##### 2.2 WebClient-backed load-balancing

- 优先使用之前使用过的实例

  ```java
  public class CustomLoadBalancerConfiguration {
  
      @Bean
      public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
              ConfigurableApplicationContext context) {
          return ServiceInstanceListSupplier.builder()
                      .withDiscoveryClient()
                      .withSameInstancePreference()
                      .build(context);
          }
      }
  }
  ```

  ```java
  @Configuration
  @LoadBalancerClients({
  	@LoadBalancerClient(value = "stock-service", configuration = CustomLoadBalancerConfiguration.class) })
  @EnableDiscoveryClient
  public class MyConfiguration {
  
      @Bean
      @LoadBalanced
      RestTemplate restTemplate(RestTemplateBuilder builder) {
  	return builder.build();
      }
  }
  ```

### 3.5 Open Feign

Feign 是 Netflix 开发的声明式、模版化的 HTTP 客户端，可以像调用本地方法一样调用远程接口。它像 Dubbo 一样，消费者直接调用接口方法，而不需要 HTTP 请求解析数据。

Spring Cloud openfeign 是对 Feign 进行的增强，支持 Spring MVC 的原生注解，自动集成 Spring Cloud 的负载均衡。

#### 1.环境配置

- 引入依赖

  在消费者服务 order 的 pom 文件中引入如下依赖

  ```xml
  <dependencies>
    	<dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-starter-openfeign</artifactId>
      </dependency>
    	<dependency>
       	<groupId>org.springframework.cloud</groupId>
       	<artifactId>spring-cloud-starter-loadbalancer</artifactId>
    	</dependency>
    	<dependency>
        	<groupId>com.alibaba.cloud</groupId>
        	<artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    	</dependency>
  </dependencies>
  ```

- 创建远程API的接口

  ```java
  /**
   * @name 指定远程接口的服务名
   * @path 指定调用 rest 接口所在的 controller 指定的 RequestMapping
   */
  @FeignClient(name = "stock-service", path = "/stock")
  public interface StockFeignService {
  
      // 指定调用的远程方法
      @GetMapping("/reduct")
      String reduct();
  }
  ```

- 在配置类上开启`Feign`

  ```java
  @Configuration
  @LoadBalancerClients({
  	@LoadBalancerClient(value = "stock-service", configuration = CustomLoadBalancerConfiguration.class) })
  @EnableDiscoveryClient
  // 必须指定包名，否则扫描当前配置类的包下
  @EnableFeignClients(basePackages = "com.youyi.zhao.feign")
  public class MyConfiguration {
  
  }
  ```

- 移除`restTemplate`，直接调用本地方法

  ```java
  @RestController
  @RequestMapping("/order")
  public class OrderController {
  
      @Autowired
      StockFeignService stockFeignService;
  
      @GetMapping("/add")
      public String add() {
  	return stockFeignService.reduct();
      }
  }
  ```

#### 2.自定义配置

##### 2.1 日志配置

4 种日志等级：

- `NONE`（默认）：不记录任何日志，适用于生产
- `BASIC`：仅记录请求方法、URL、响应状态代码以及执行时间，适用于生产环境问题追踪
- `HEADERS`：记录 BASIC 级别的基础上，记录请求和相应的 header
- `FULL`：记录请求和响应的 header、body 和元数据，适用于开发测试环境问题定位

**使用**

- 注册一个日志 bean，指定日志级别

  - 配置类方式

    ```java
    @Configuration
    public class FeignConfig {
        @Bean
        Logger.Level feignLoggerLevel() {
            return Logger.Level.FULL;
        }
    }
    ```

    > **注意**
    >
    > 这样默认是全局配置，如果希望对某一个服务进行配置，则移除`@Configuration`注解，并在`@FeignClient`注解中添加`configuration`属性指定配置类。
    >
    > ```java
    > @FeignClient(name = "stock-service", path = "/stock", configuration = FeignConfig.class)
    > ```

  - 配置文件方式

    ```yaml
    feign:
    	client:
    		config:
    			stock-service:
    				loggerLevel: BASIC
    ```

- 修改输出日志级别

  ```yaml
  logging:
  	level:
  		com.youyi.zhao.feign: debug
  ```

##### 2.2 契约配置

Spring Cloud 对 Feign 进行了扩展，支持使用 Spring MVC 的注解，例如`@RequestMapping`。原生的 Feign 是不支持 Spring MVC 注解的，如果想使用原生注解，就需要使用契约配置，Spring Cloud 中默认的是`SpringMvcContact`。用于升级旧版本的代码。

- 配置类方式

  ```java
  @Bean
  public Contract feignContract() {
      return new Contact.Default();
  }
  ```

- 配置文件方式

  ```yaml
  feign:
  	client:
  		config:
  			stock-service:
  				contact: feign.Contract.Default
  ```

此时就不能在 Feign 中使用 Spring MVC 的注解，例如`@RequestMapping`要替换成`@RequestLine`。

##### 2.3 超时时间配置

通过`Options`可以配置连接超时时间和读取超时时间，Options 的第一个参数是连接的超时时间（ms），默认 2000ms，第二个是请求处理的超时时间（ms），默认 5000ms。不设置时使用负载均衡的配置。

- 配置类方式

  ```java
  @Bean
  public Request.Options options() {
      return new Request.Options(5000, 10000);
  }
  ```

- 配置文件方式

  ```yaml
  feign:
  	client:
  		config:
  			stock-service:
  				connectTimeout: 5000
  				readTimeout: 10000
  ```

超时会抛出`java.net.SocketTimeoutException`异常。

##### 2.4 自定义拦截器

配置消费者调用提供者时的拦截器，区别于 Spring MVC 的拦截器是用于客户端调用消费者。

```java
public class FeignAuthRequestInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        String access_token = UUID.randomUUID().toString();
        template.header("auth", access_token);
        template.query("id", "123");
    }
}

// 注册到容器中
@Bean
public FeignAuthRequestInterceptor feignAuthRequestInterceptor() {
    return new FeignAuthRequestInterceptor();
}
```

或通过配置文件注册到容器中

```yaml
feign:
	client:
		config:
			stock-service:
				requestInterceptors[0]:
					com.youyi.zhao.feign.interceptor.FeignAuthRequestInterceptor
```

### 3.6 配置中心

Nacos 提供用于存储配置和其他元数据的 key-value 存储，为分布式系统中的外部化配置提供服务器端和客户端支持。

参考：https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-config

与 Spring Cloud Config 的对比：

- Spring Cloud Config 大部分场景需要结合 GIT 使用，动态变更还需要依赖 Spring Cloud Bus 消息总线来通知所有的客户端变化，且不提供可视化界面
- Nacos Config 使用长轮询更新配置，速度上比 Spring Cloud Config 快得多

#### 1.权限管理

启动权限：修改 `nacos/conf/application.properties`

```properties
nacos.core.auth.enabled=true
```

或者在容器启动时指定`-e NACOS_AUTH_ENABLE=true`

#### 2.配置管理

- “配置列表” - “新建配置”
- 指定“Data ID”，通常采用包名来保证全局唯一性
- 指定“Group”，进行分组
- 指定配置格式，例如“properties”
- 添加 key-value
- 保存

最终会保存在“config_info”数据库中，可以通过历史版本进行回滚，通过 clone 拷贝到其他命名空间。

#### 3.客户端读取配置

添加依赖`spring-cloud-starter-alibaba-nacos-config`

```xml
 <dependencies>
  	<dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
  	<dependency>
     	<groupId>org.springframework.cloud</groupId>
     	<artifactId>spring-cloud-starter-loadbalancer</artifactId>
  	</dependency>
  	<dependency>
      	<groupId>com.alibaba.cloud</groupId>
      	<artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
  	</dependency>
  	<dependency>
      	<groupId>com.alibaba.cloud</groupId>
      	<artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
  	</dependency>
</dependencies>
```

