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

## 3.Spring Cloud Alibaba简介

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

