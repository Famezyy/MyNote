# 第02章_Nacos与OpenFeign

## 1.案例：升级旧微服务架构

### 1.1 旧架构使用`ip`访问

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

### 1.2 使用SpringCloudAlibaba

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

## 2.注册中心

### 2.1 注册中心简介

#### 1. 演变过程

注册中心用来管理所有微服务、解决微服务之间调用关系错综复杂、难以维护的问题。

1. 将远程调用地址写在代码中，缺点：不易于更新维护

2. 手动维护一份服务地址的注册表，远程调用时去注册表中找相应的地址，缺点：不易于服务的水平扩展

3. 利用 nginx 维护服务地址的注册表，从而实现水平扩展时的负载均衡，缺点：微服务数量较多时配置维护困难

4. 利用注册中心动态维护注册表

   - 各个服务启动时注册服务到注册中心
   - 每次调用服务时从注册中心获取服务列表进行调用 

   缺点：无法监控服务是否宕机。

5. 引入心跳机制，定时监测服务是否运行

#### 2.主流的注册中心

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

#### 3.Nacos Discovery

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

###  2.2 Nacos Server部署

| Spring Cloud Alibaba Version | Sentinel Version | Nacos Version | RocketMQ Version | Dubbo Version | Seata Version |
| ---------------------------- | ---------------- | ------------- | ---------------- | ------------- | ------------- |
| 2.2.10-RC1                   | 1.8.6            | 2.2.0         | 4.9.4            | ~             | 1.6.1         |
| 2022.0.0.0-RC1               | 1.8.6            | 2.2.1-RC      | 4.9.4            | ~             | 1.6.1         |
| 2.2.9.RELEASE                | 1.8.5            | 2.1.0         | 4.9.4            | ~             | 1.5.2         |
| 2021.0.4.0                   | 1.8.5            | 2.0.4         | 4.9.4            | ~             | 1.5.2         |

根据版本依赖关系，这里使用`2.0.4`版本。

#### 1.安装

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

  > **提示**
  >
  > 需要提前安装`java`和`maven`。

- 下载编译后压缩包方式

  从[最新稳定版本](https://github.com/alibaba/nacos/releases)下载`nacos-server-$version.tar.gz`包。

  ```bash
  unzip nacos-server-$version.zip
  # or tar -xvf nacos-server-$version.tar.gz
  cd nacos/bin
  ```

#### 2.运行

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
> start) sh $NACOS/bin/startup.sh -m standalone;;
> stop) sh $NACOS/bin/shutdown.sh;;
> *) echo "require start|stop";;
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

#### 3.登陆

等服务器启动成功后，查看输出日志，访问指定地址，用户名和密码默认是`nacos`。

> **注意**
>
> 需要打开防火墙指定端口或关闭防火墙。

#### 4.关闭

- Linux/Unix/Mac：`sh shutdown.sh`
- Wubdiws：`shutdown.cmd`

#### 5.集群启动

集群启动需要数据源，可以使用 mySQL。

**（1）配置集群配置文件**

在 nacos 的解压目录的 conf 目录下，有配置文件 cluster.conf，请每行配置成 ip:port。（请配置 3 个或 3 个以上节点）

```plain
# ip:port
200.8.9.16:8848
200.8.9.17:8848
200.8.9.18:8848
```

**（2）初始化 MySQL 数据库**

[sql语句源文件](https://github.com/alibaba/nacos/blob/master/distribution/conf/mysql-schema.sql)

nacos 2.2.0 之前：https://raw.githubusercontent.com/alibaba/nacos/1.0.0-RC3/distribution/conf/nacos-mysql.sql

在 mysql 中执行：

```sql
create database nacos_config;
use nacos_config;
source /tmp/nacos-mysql.sql;
```

创建 nacos 用户：

```sql
CREATE USER 'nacos'@'%' IDENTIFIED BY 'nacos';
grant all privileges on nacos_config.* to 'nacos'@'%';
# 8.0 下执行
# ALTER USER 'nacos'@'%' IDENTIFIED WITH mysql_native_password BY 'nacos';
```

**（3）application.properties 配置**

修改数据库连接 url，用户名和密码可使用`nacos`。参考：[application.properties配置文件](https://github.com/alibaba/nacos/blob/master/distribution/conf/application.properties)

**（4）启动**

实际生产中可配合 keepalived + LVS 进行使用。

#### 6.调整日志输出级别

```bash
# 调整 naming 模块的 naming-raft.log 的级别为 error
curl -X PUT '$nacos_server:8848/nacos/v1/ns/operator/log?logName=naming-raft&logLevel=error'
# 调整 config 模块的 config-dump.log 的级别为 warn
curl -X PUT '$nacos_server:8848/nacos/v1/cs/ops/log?logName=config-dump&logLevel=warn'
```

### 2.3 Nacos Client部署

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
          # 配置了 nacos 的 nacos.core.auth.enabled=false 则可以不需要用户名和密码
          # username: nacos
          # password: nacos
          # namespace: public 命名空间，默认 public，不是 public 时需要指定 hash ID 而不是名称
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
      name: order-service
    cloud:
      nacos:
        server-addr: 192.168.11.100:8848
        discovery:
          # username: nacos
          # password: nacos
          namespace: public
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

### 2.4 Nacos的配置界面

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

### 2.5 Nacos的配置项

更多关于 spring-cloud-starter-alibaba-nacos-discovery 的 starter 配置项：https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-discovery

### 2.6 Docker下集群模式部署

这里为了简便，在`192.168.11.100`上的 Docker 中部署三个 Nacos 服务

- 创建 MySQL 容器并初始化数据库

  创建容器参考[Docker 部署 Mysql](../../../云原生/Docker/第03章_Docker部署.md#52-Mysql)，创建表：[sql语句源文件](https://github.com/alibaba/nacos/blob/master/distribution/conf/mysql-schema.sql)

  > **注意**
  >
  > nacos.2.1.0 及之前数据库初始化脚本为 nacos-mysql.sql，2.2.0 之后重命名为 mysql-schema.sql

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
      	server nacos01:9849;
      	server nacos02:9849;
      	server nacos03:9849;
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

### 2.7 Docker Compose创建集群

参考：https://nacos.io/zh-cn/docs/quick-start-docker.html

**目录构成**

```bash
.
├── mysql
│   ├── Dockerfile
│   └── mysql.env
├── nacos
│   └── nacos.env
├── nacos-server.yaml
└── nginx
    ├── conf.d
    │   └── default.conf
    ├── Dockerfile
    └── nginx.conf
```

> **提示**
>
> 需要安装`yum install -y tree`工具。

**docker compose**

```yaml
version: "3.8"

services:
  mysql:
    build:
      context: ./
      dockerfile: mysql/Dockerfile
    # 注意 image 不能存在
    image: my_mysql
    container_name: mysql
    ports:
      - "3306:3306"
    volumes:
      - /youyi/mysql_master/log:/var/log/mysql
      - /youyi/mysql_master/data:/var/lib/mysql
      - /youyi/mysql_master/conf:/etc/mysql/conf.d
    env_file:
      - ./mysql/mysql.env
    restart: always 
    networks:
      - my_net
    healthcheck:
      test: [ "CMD", "mysqladmin" ,"ping", "-h", "localhost" ]
      interval: 5s
      timeout: 10s
      retries: 10
  
  nacos1:
    hostname: nacos1
    container_name: nacos1
    image: nacos/nacos-server:${NACOS_VERSION}
    volumes:
      - /youyi/nacos/nacos1:/home/nacos/logs
    env_file:
      - ./nacos/nacos.env
    restart: always
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - my_net
    healthcheck:
      test: [ "CMD", "curl", "-f", "nacos1:8848/nacos" ]
      interval: 5s
      timeout: 10s
      retries: 10
        
  nacos2:
    hostname: nacos2
    container_name: nacos2
    image: nacos/nacos-server:${NACOS_VERSION}
    volumes:
      - /youyi/nacos/nacos2:/home/nacos/logs
    env_file:
      - ./nacos/nacos.env
    restart: always
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - my_net
    healthcheck:
      test: [ "CMD", "curl", "-f", "nacos2:8848/nacos" ]
      interval: 5s
      timeout: 10s
      retries: 10
    
  nacos3:
    hostname: nacos3
    container_name: nacos3
    image: nacos/nacos-server:${NACOS_VERSION}
    volumes:
      - /youyi/nacos/nacos3:/home/nacos/logs
    env_file:
      - ./nacos/nacos.env
    restart: always
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - my_net
    healthcheck:
      test: [ "CMD", "curl", "-f", "nacos3:8848/nacos" ]
      interval: 5s
      timeout: 10s
      retries: 10
        
  nginx:
    build:
      context: ./
      dockerfile: nginx/Dockerfile
    image: my_nginx
    hostname: nginx
    container_name: nginx
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - /youyi/nginx/html:/etc/nginx/html
      - /youyi/nginx/log:/var/log/nginx
    ports:
      - "8848:8848"
      - "9848:9848"
      - "9555:9555"
    networks:
      - my_net
    depends_on:
      nacos1:
        condition: service_healthy
      nacos2:
        condition: service_healthy
      nacos3:
        condition: service_healthy

networks:
  my_net:
    name: my_net
    driver: bridge
```

**`./mysql/Dockerfile`**

```dockerfile
FROM mysql:5.7.40
ADD https://raw.githubusercontent.com/alibaba/nacos/1.0.0-RC3/distribution/conf/nacos-mysql.sql /docker-entrypoint-initdb.d/nacos-mysql.sql
# 适用于 nacos:v2.2.0 及之后的版本：ADD https://raw.githubusercontent.com/alibaba/nacos/develop/distribution/conf/mysql-schema.sql /docker-entrypoint-initdb.d/nacos-mysql.sql
RUN chown -R mysql:mysql /docker-entrypoint-initdb.d/nacos-mysql.sql
EXPOSE 3306
CMD ["mysqld", "--character-set-server=utf8mb4", "--collation-server=utf8mb4_unicode_ci"]
```

**`./mysql/mysql.env`**

```properties
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=nacos_devtest
MYSQL_USER=nacos
MYSQL_PASSWORD=nacos
```

**`./nacos/nacos.env`**

```properties
PREFER_HOST_MODE=hostname
NACOS_SERVERS=nacos1:8848 nacos2:8848 nacos3:8848
MYSQL_SERVICE_HOST=192.168.11.100
MYSQL_SERVICE_DB_NAME=nacos_devtest
MYSQL_SERVICE_PORT=3306
MYSQL_SERVICE_USER=nacos
MYSQL_SERVICE_PASSWORD=nacos
mYSQL_SERVICE_DB_PARAM=characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false&allowPublicKeyRetrieval=true
NACOS_AUTH_ENABLE=true
```

**`./.env`**

```properties
NACOS_VERSION=v2.0.4
MYSQL_VERSION=5.7
```

**`./nginx/conf.d/default.conf`**

```bash
upstream nacoscluster-8848 {
    server nacos1:8848;
    server nacos2:8848;
    server nacos3:8848;
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

**`./nginx/Dockerfile`**

```dockerfile
FROM nginx
ADD ./nginx/nginx.conf /etc/nginx/nginx.conf
```

**`./nginx/nginx.conf`**

```bash
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}

# 在 http 同级下添加三层负载均衡
stream {
        upstream nacoscluster-9848 {
        server nacos1:9848;
        server nacos2:9848;
        server nacos3:9848;
        }
        server {
                listen 9848;
                proxy_pass nacoscluster-9848;
        }
    
        upstream nacoscluster-9849 {
        server nacos1:9849;
        server nacos2:9849;
        server nacos3:9849;
        }
        server {
                listen 9849;
                proxy_pass nacoscluster-9849;
        }
}
```

**执行**

```bash
docker compose -f nacos-server.yaml up -d
```

> **注意**
>
> 如果出现监听端口已占用异常，则执行：
>
> ```bash
> netstat -nlp | grep LISTEN
> kill pid
> ```

## 3.负载均衡

目前主流的负载方案：

- 服务端负载均衡：在消费者和服务提供方中间使用独立的代理方式进行负载，分为硬件实现（比如 F5）和软件实现（比如 Nginx）
- 客户端负载均衡：客户端根据请求做负载均衡

**常见的负载均衡算法**

- 随机
- 轮询
- 加权轮询
- 地址哈希
- 最小链接数

### 3.1 Ribbon（deprecated）

> **注意**
>
> Spring Cloud 2020 版本以后，默认移除了对 Netflix 的依赖，其中就包括 Ribbon，官方默认推荐使用 Spring Cloud Loadbalancer 正式替换 Ribbon，并成为了 Spring Cloud 负载均衡器的唯一实现。

Ribbon 是一种基于 Netflix Ribbon 实现的客户端负载均衡。通过 Load Balance 获取到服务提供的所有机器实例并在客户端进行维护，Ribbon 会基于某种规则（轮询、随机）去调用这些服务。Ribbon 也可以实现自己的负载均衡算法。

#### 1.常用接口

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

#### 2.代码示例

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

#### 3.自定义负载均衡

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

#### 4.开启饥饿加载

负载均衡器默认是懒加载的，只有在第一次使用到时才会加载，可通过以下设置改成饥饿加载：

```yaml
ribbon:
	eager-load:
		# 开启饥饿加载
		enabled: true
		# 指定需要饥饿加载的服务名称
		clients: stock-service
```

### 3.2 LoadBalancer

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

#### 1.修改负载均衡策略

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

#### 2.WebClient-backed load-balancing

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

## 4.Open Feign

Feign 是 Netflix 开发的声明式、模版化的 HTTP 客户端，可以像调用本地方法一样调用远程接口。它像 Dubbo 一样，消费者直接调用接口方法，而不需要 HTTP 请求解析数据。

Spring Cloud openfeign 是对 Feign 进行的增强，支持 Spring MVC 的原生注解，自动集成 Spring Cloud 的负载均衡。

==使用 Feign 时需要引入`LoadBalancer`的依赖！==

### 4.1 环境配置

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
      
      // 传参时一定要加上@RequestParam
      @GetMapping("/reduct")
      String reduct(@RequestParam("productId") Integer productId);
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

### 4.2 自定义配置

#### 1.日志配置

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

#### 2.契约配置

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

#### 3.超时时间配置

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

#### 4.自定义拦截器

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

## 5.配置中心

Nacos 提供用于存储配置和其他元数据的 key-value 存储，为分布式系统中的外部化配置提供服务器端和客户端支持。

参考：https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-config

与 Spring Cloud Config 的对比：

- Spring Cloud Config 大部分场景需要结合 GIT 使用，动态变更还需要依赖 Spring Cloud Bus 消息总线来通知所有的客户端变化，且不提供可视化界面
- Nacos Config 使用长轮询更新配置，速度上比 Spring Cloud Config 快得多

### 5.1 权限管理

参考：https://nacos.io/zh-cn/docs/auth.html

启动权限：修改 `nacos/conf/application.properties`

```properties
nacos.core.auth.enabled=true
```

或者在容器启动时指定环境变量`NACOS_AUTH_ENABLE=true`，开启权限后，就需要在消费端和提供端声明 nacos 的用户名和密码。

### 5.2 配置管理

> **注意**
>
> 使用集群时，数据库的表结构要与 nacos 版本一致。

- “配置列表” - “新建配置”
- 选择命名空间，最好按环境进行分配
- 指定“Data ID”，最好按工程进行分配
- 指定“Group”，进行分组，最好按项目进行分配
- 指定配置格式，例如“properties”
- 添加 key-value
- 保存

最终会保存在“config_info”数据库中，可以通过历史版本进行回滚，通过 clone 拷贝到其他命名空间。

### 5.3 客户端读取配置

首先在 Nacos 添加如下配置：

```
Data ID:    nacos-config.properties

Group  :    DEFAULT_GROUP

配置格式:    Properties

配置内容：   user.id=zhao
            user.age=90
```

#### 1.使用bootStrap.properties

- 添加依赖`spring-cloud-starter-alibaba-nacos-config`

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
      <!-- 在SpringCloud 2020.* 版本把加载 bootstrap 的功能禁用了，导致在读取文件的时候读取不到而报错，所以我们只要把bootstrap从新导入进来就会生效了 -->
      <dependency>
  	    <groupId>org.springframework.cloud</groupId>
  	    <artifactId>spring-cloud-starter-bootstrap</artifactId>
  	</dependency>
  </dependencies>
  ```

- 使用`bootstrap.yaml`配置文件来配置 conf，例如：

  ```yaml
  spring:
    application:
      name: nacos-config
    cloud:
      nacos:
        server-addr: 192.168.11.100:8848
        # username: nacos
        # password: nacos
        conf:
        # file-extension: properties 默认 properties，使用其他格式时必须指定，例如 yaml
  ```

  > **注意**
  >
  > 此时会默认查找服务名`nacos-config`和服务名 + yaml 结尾的`nacos-config.yaml`配置文件。其他普通配置都可放在 nacos 中（例如数据库配置）。


#### 2.使用application.properties（推荐）

也可以在中使用`spring.config.import`加载配置文件：

```properties
spring.application.name=order-server
spring.cloud.nacos.server-addr=192.168.11.100:8848
spring.cloud.nacos.username=nacos
spring.cloud.nacos.password=nacos
# 可以使用${}来读取 JVM 参数来区分环境
spring.config.import=nacos:order-server-${env}
```

- 添加多个配置文件时用`,`隔开，前面的会覆盖后面的配置，如`nacos:order-${env},nacos:common-prop`

- 此时同样可以按 namespace、Group、DataId 获取配置。但是==不能和[自定义获取配置](#5.4-自定义DataId获取配置)配合使用==

### 5.4 获取配置

通过`env.getProperties()`获取

```java
@RestController
@RequestMapping("/order")
public class OrderController {

    @Autowired
    Environment env;

    @GetMapping("/add")
    public String add() {
	return env.getProperty("user.id") + env.getProperty("user.age");
    }
}
```

通过注解获取

```java
@NacosPropertySource(dataId = "pre-loan-provider-nacos",groupId = "pre-loan",autoRefreshed = true)
```

#### 1.按环境获取配置

此时只能使用`spring.application.name`作为文件名来获取，不能使用上节的方式。

spring-cloud-starter-alibaba-nacos-config 在加载配置的时候，不仅仅加载了以 dataId 为 `${spring.application.name}.${file-extension:properties}` 为前缀的基础配置，还加载了 dataId 为 `${spring.application.name}-${profile}.${file-extension:properties}` 的基础配置。在日常开发中如果遇到多套环境下的不同配置，可以通过Spring 提供的`${spring.profiles.active}`这个配置项来配置。

```yaml
spring:
  profiles:
    active:
    - pro  # 修改为 dev 即可切换配置源
  application:
    name: nacos-config
  cloud:
    nacos:
      server-addr: 192.168.11.100:8848
      username: nacos
      password: nacos
      config:
        file-extension: yaml
```

> **注意**
>
> `${spring.profiles.active}`当通过配置文件来指定时必须放在 bootstrap.properties 文件中。

Nacos 上新增一个 dataId 为：nacos-config-develop.yaml 的基础配置，如下所示：

```
Data ID:        nacos-config-dev.yaml

Group  :        DEFAULT_GROUP

配置格式:        YAML

配置内容：        user.id=zhao
                user.age=90
```

如果需要切换到生产环境，只需要更改`${spring.profiles.active}`参数配置即可。如下所示：

```
spring.profiles.active=pro
```

同时生产环境上 Nacos 需要添加对应 dataId 的基础配置。例如，在生成环境下的 Nacos 添加了 dataId 为：nacos-config-pro.yaml 的配置：

```
Data ID:        nacos-config-pro

Group  :        DEFAULT_GROUP

配置格式:        YAML

配置内容：       user.id=zhao
                user.age=50
```

> **注意**
>
> 此案例中我们通过`spring.profiles.active=<profilename>`的方式写死在配置文件中，而在真正的项目实施过程中这个变量的值是需要不同环境而有不同的值。这个时候通常的做法是通过`-Dspring.profiles.active=<profile>`参数指定其配置来达到环境间灵活的切换。

#### 2.按namespace获取配置

实际更多使用的是**命名空间**来区分。

```yaml
spring:
  application:
    name: nacos-config
  cloud:
    nacos:
      server-addr: 192.168.11.100:8848
      username: nacos
      password: nacos
      config:
        file-extension: yaml
        namespace: 40ae7c47-648d-4082-b7d1-eb85ddc0222f
```

> **注意**
>
> 配置`spring.cloud.nacos.config.namespace`必须放在 bootstrap.properties 文件中。默认使用 public 作为命名空间，指定`spring.cloud.nacos.config.namespace`的值时必须==**是 namespace 对应的 id**==，id 值可以在 Nacos 的控制台获取。并且在添加配置时注意不要选择其他的 namespace，否则将会导致读取不到正确的配置。

#### 3.按Group获取配置

在没有明确指定`${spring.cloud.nacos.config.group}`配置的情况下， 默认使用的是 DEFAULT_GROUP。如果需要自定义自己的 Group，可以通过以下配置来实现：

```yaml
spring:
  cloud:
    nacos:
      config:
        group: DEVELOP_GROUP
```

> **注意**
>
> 该配置必须放在 bootstrap.properties 文件中。并且在添加配置时 Group 的值一定要和`spring.cloud.nacos.config.group`的配置值一致。

### 5.4 自定义DataId获取配置

`file-extension`、`group`仅适用于`${spring.application.name}`和`${spring.config.import}`的配置文件，不适用于自定义；`namespace`也适用于自定义。

#### 1.shared-configs

在 Nacos 配置如下

```bash
Data ID:        customized-id.yaml

Group  :        DEFAULT_GROUP

配置格式:        YAML

配置内容：       user.id=zhao
                user.age=50
```

boostrap.yaml 配置如下

```yaml
spring:
  application:
    name: nacos-config
  cloud:
    nacos:
      server-addr: 192.168.11.100:8848
      username: nacos
      password: nacos
      config:
        shared-configs:
        - data-id: customized-id.yaml
          refresh: true # 默认 false
        # group: 默认组
```

> **注意**
>
> 多个 Data Id 同时配置时，他的优先级关系是`spring.cloud.nacos.config.extension-configs[n].data-id`其中 n 的值越大，优先级越高。
>
>  `spring.cloud.nacos.config.extension-configs[n].data-id`的值必须带文件扩展名，文件扩展名既可支持 properties，又可以支持 yaml/yml。 此时`spring.cloud.nacos.config.file-extension`的配置对自定义扩展配置的 Data Id 文件扩展名没有影响。

#### 2.extension-configs

在 Nacos 配置如下

```bash
Data ID:        customized-id.yaml

Group  :        DEFAULT_GROUP

配置格式:        YAML

配置内容：       user.id=zhao
                user.age=50
```

boostrap.yaml 配置如下

```yaml
spring:
  application:
    name: nacos-config
  cloud:
    nacos:
      server-addr: 192.168.11.100:8848
      username: nacos
      password: nacos
      config:
        extension-configs:
        - data-id: customized-id.yaml
          refresh: true # 默认 false，即不会自动更新值
        # group: 默认组
```

### 5.5 配置的优先级

1. 如果配置了 profile，则 profile > 默认配置文件，如果在 profile 未读取到则会继续读取默认配置文件

2. 没有其他配置时，默认会拉取`${spring.application.name}`和`${spring.application.name}.properties`的配置，后者有较高优先级

3. 通过`spring.cloud.nacos.config.extension-configs[n].data-id`的方式支持多个扩展 Data Id 的配置
4. 通过`spring.cloud.nacos.config.shared-configs[n].data-id`支持多个共享 Data Id 的配置

总结：profile > 默认 > extension > shared。

一般来讲，shared 用于公共的，默认用于各个服务的，extension 用于临时修改公共配置。

### 5.6 与注册中心结合

可以将配置中心的配置放在`bootstrap.yaml`中，`$spring.application.name`一定要放在`bootstrap.yaml`中。

```yaml
spring:
  application:
    name: order-service
  cloud:
    nacos:
      server-addr: 192.168.11.100:8848
      # username: nacos
      # password: nacos
      config:
        namespace: 40ae7c47-648d-4082-b7d1-eb85ddc0222f
        # file-extension: yaml
        extension-configs:
        - data-id: customized-id.yaml
          refresh: true
      discovery:
        namespace: 40ae7c47-648d-4082-b7d1-eb85ddc0222f
```

此时会拉取以下三个`dataId`的配置文件，优先级依次递减

- `dataId=order-service.properties`
- `dataId=order-service`
- `dataId=customized-id.yaml`

### 5.7 @RefreshScope

`@value`无法动态感知变更，需要在类上标注`@RefreshScope`

```java
@RestController
@RequestMapping("/order")
@RefreshScope
public class OrderController {

    @Value("${user.id}")
    String id;

    @GetMapping("/add")
    public String add() {
	return id;
    }
}
```
