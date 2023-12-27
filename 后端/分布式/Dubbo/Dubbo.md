<a style="float:right" href="https://dubbo.apache.org/zh/docs/advanced/">参考</a>

## 一、基础知识

### 1.分布式基础理论

#### 1.1 什么是分布式系统？

《分布式系统原理与范型》定义：分布式系统是若干独立计算机的集合，这些计算机对于用户来说就像单个相关系统。

分布式系统（distributed system）是建立在网络之上的软件系统。

随着互联网的发展，网站应用的规模不断扩大，常规的垂直应用架构已无法应对，分布式服务架构以及流动计算架构势在必行，亟需**一个治理系统**确保架构有条不紊的演进。

#### 1.2 发展演变

<img src="img\image-20220129231551683.png" alt="image-20220129231551683"  />

**单一应用架构**

当网站流量很小时，只需一个应用，将所有功能都部署在一起，以减少部署节点和成本。此时，用于简化增删改查工作量的数据访问框架 (ORM) 是关键。

<img src="img\image-20220129231559329.png" alt="image-20220129231559329"  />

适用于小型网站，小型管理系统，将所有功能都部署到一个功能里，简单易用。

缺点：

- 性能扩展比较难
- 协同开发问题
- 不利于升级维护

**垂直应用架构**

当访问量逐渐增大，单一应用增加机器带来的加速度越来越小，将应用拆成互不相干的几个应用，以提升效率。此时，用于加速前端页面开发的 Web 框架 (MVC) 是关键。

<img src="img\image-20220129231612515.png" alt="image-20220129231612515"  />

通过切分业务来实现各个模块独立部署，降低了维护和部署的难度，团队各司其职更易管理，性能扩展也更方便，更有针对性。

缺点：

- 公用模块无法重复利用，开发性的浪费

**分布式服务架构**

​    当垂直应用越来越多，应用之间交互不可避免，将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心，使前端应用能更快速的响应多变的市场需求。此时，用于提高业务复用及整合的分布式服务框架 (RPC) 是关键。

<img src="img\image-20220129231635193.png" alt="image-20220129231635193"  />

**流动计算架构**

​    当服务越来越多，容量的评估，小服务资源的浪费等问题逐渐显现，此时需增加一个调度中心基于访问压力实时管理集群容量，提高集群利用率。此时，用于提高机器利用率的资源调度和治理中心(SOA)[Service Oriented Architecture] 是关键。

<img src="img\image-20220129231640958.png" alt="image-20220129231640958"  />

#### 1.3 RPC

RPC【Remote Procedure Call】是指远程过程调用，是一种进程间通信方式，他是一种技术的思想，而不是规范。它允许程序调用另一个地址空间（通常是共享网络的另一台机器上）的过程或函数，而不用程序员显式编码这个远程调用的细节。即程序员无论是调用本地的还是远程的函数，本质上编写的调用代码基本相同。

#### 1.4 RPC 基本原理

<img src="img\image-20220129231653103.png" alt="image-20220129231653103"  />

<img src="img\image-20220129231705255.png" alt="image-20220129231705255"  />

RPC 两个核心模块：

- 通讯
- 序列化

### 2.dubbo 核心概念

#### 2.1 简介

Apache Dubbo 是一款高性能、轻量级的开源 Java RPC 框架，它提供了三大核心能力：面向接口的远程方法调用，智能容错和负载均衡，以及服务自动注册和发现。

官网：http://dubbo.apache.org/

#### 2.2 基本概念

<img src="img\image-20220129231537961.png" alt="image-20220129231537961"  />

**服务提供者（Provider**）：暴露服务的服务提供方，服务提供者在启动时，向注册中心注册自己提供的服务。

**服务消费者（Consumer**）: 调用远程服务的服务消费方，服务消费者在启动时，向注册中心订阅自己所需的服务，服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。

**注册中心（Registry**）：注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。

**监控中心（Monitor**）：服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。

调用关系说明：

- 服务容器负责启动，加载，运行服务提供者
- 服务提供者在启动时，向注册中心注册自己提供的服务
- 服务消费者在启动时，向注册中心订阅自己所需的服务
- 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者
- 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用
- 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心

<img src="img\image-20220419220001821.png" alt="image-20220419220001821" style="zoom:80%;" />

### 3.dubbo环境搭建

#### 3.1 【windows】- 安装zookeeper

| 1、下载zookeeper<br />网址 https://archive.apache.org/dist/zookeeper/zookeeper-3.4.13/ |
| ------------------------------------------------------------ |
| 2、解压zookeeper<br />解压运行zkServer.cmd ，初次运行会报错，没有 zoo.cfg 配置文件 |
| 3、修改 zoo.cfg 配置文件<br />将 conf 下的 zoo_sample.cfg 复制一份改名为 zoo.cfg 即可。<br />注意几个重要位置：<br />dataDir=./  临时数据存储的目录（可写相对路径）<br />clientPort=2181  zookeeper 的端口号<br />修改完成后再次启动zookeeper |
| 4、使用zkCli.cmd测试<br />ls /：列出zookeeper根下保存的所有节点<br />create –e /atguigu 123：创建一个atguigu节点，值为123<br />get /atguigu：获取/atguigu节点的值 |

#### 3.2 【windows】- 安装 dubbo-admin 管理控制台

dubbo 本身并不是一个服务软件。它其实就是一个 jar 包能够帮你的 java 程序连接到 zookeeper，并利用 zookeeper 消费、提供服务。所以你不用在 Linux 上启动什么 dubbo 服务。

但是为了让用户更好的管理监控众多的 dubbo 服务，官方提供了一个可视化的监控程序，不过这个监控即使不装也不影响使用。

| 1、下载dubbo-admin  https://github.com/apache/incubator-dubbo-ops <br /><img src="img\image-20220129232129444.png" alt="image-20220129232129444"  /> |
| ------------------------------------------------------------ |
| 2、进入目录，修改 dubbo-admin 配置 <br />修改 src\main\resources\application.properties  指定zookeeper地址 |
| 3、打包 dubbo-admin <br />mvn clean package -Dmaven.test.skip=true |
| 4、运行dubbo-admin<br />java -jar dubbo-admin-0.0.1-SNAPSHOT.jar<br />**注意：【有可能控制台看着启动了，但是网页打不开，需要在控制台按下ctrl+c**即可】<br />默认使用 root/root 登陆<br /><img src="img\image-20220129232208689.png" alt="image-20220129232208689"  /> |

#### 3.3 【linux】- 安装zookeeper

- 安装jdk

  | 1、下载jdk  http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html <br />不要使用 wget 命令获取 jdk 链接，这是默认不同意，导致下载来的 jdk 压缩内容错误 |
  | ------------------------------------------------------------ |
  | 2、上传到服务器并解压<br /><img src="img\image-20220129232346897.png" alt="image-20220129232346897"  /> |
  | 3、设置环境变量<br />/usr/local/java/jdk1.8.0_171<br /><img src="img\image-20220129232402465.png" alt="image-20220129232402465"  /><br />文件末尾加入下面配置<br />export JAVA_HOME=/usr/local/java/jdk1.8.0_171<br />export JRE_HOME=\${JAVA_HOME}/jre<br />export CLASSPATH=.:${JAVA_HOME}/lib:\${JRE_HOME}/lib<br />export PATH=\${JAVA_HOME}/bin:\$PATH<br /><img src="img\image-20220129232546196.png" alt="image-20220129232546196"  /> |
  | 4、使环境变量生效&测试 JDK<br /><img src="img\image-20220129232553295.png" alt="image-20220129232553295"  /> |

- 安装zookeeper

  | 1、下载zookeeper<br />网址 https://archive.apache.org/dist/zookeeper/zookeeper-3.4.11/<br />wget https://archive.apache.org/dist/zookeeper/zookeeper-3.4.11/zookeeper-3.4.11.tar.gz |
  | ------------------------------------------------------------ |
  | 2、解压<br /><img src="img\image-20220129232613992.png" alt="image-20220129232613992"  /> |
  | 3、移动到指定位置并改名为 zookeeper<br /><img src="img\image-20220129232619342.png" alt="image-20220129232619342"  /><br /><img src="img\image-20220129232628936.png" alt="image-20220129232628936"  /> |

- 开机启动 zookeeper

  **复制如下脚本**

  ```bash
  #!/bin/bash
  #chkconfig:2345 20 90
  #description:zookeeper
  #processname:zookeeper
  ZK_PATH=/usr/local/zookeeper
  export JAVA_HOME=/usr/local/java/jdk1.8.0_171
  case $1 in
  	start) sh $ZK_PATH/bin/zkServer.sh start;;
  	stop) sh $ZK_PATH/bin/zkServer.sh stop;;
  	status) sh $ZK_PATH/bin/zkServer.sh status;;
  	restart) sh $ZK_PATH/bin/zkServer.sh restart;;
  	*) echo "require start|stop|status|restart" ;;
  esac
  ```

  <img src="img\image-20220129232900137.png" alt="image-20220129232900137"  />

   **把脚本注册为 Service**

  <img src="img\image-20220129232912748.png" alt="image-20220129232912748"  />

  **增加权限**

  <img src="img\image-20220129232919860.png" alt="image-20220129232919860"  />

- 配置 zookeeper

  - 初始化 zookeeper 配置文件

    拷贝 /usr/local/zookeeper/conf/zoo_sample.cfg  

    到同一个目录下改个名字叫 zoo.cfg

    <img src="img\image-20220129232958692.png" alt="image-20220129232958692"  />

  - 启动 zookeeper

    <img src="img\image-20220129233009395.png" alt="image-20220129233009395"  />

#### 3.4 【linux】- 安装 dubbo-admin 管理控制台

- **安装Tomcat8（旧版 dubbo-admin 是 war，新版是 jar 不需要安装 Tomcat）**

  - 下载Tomcat8并解压

    https://tomcat.apache.org/download-80.cgi

    wget http://mirrors.shu.edu.cn/apache/tomcat/tomcat-8/v8.5.32/bin/apache-tomcat-8.5.32.tar.gz

  - 解压移动到指定位置

    <img src="img\image-20220129233141819.png" alt="image-20220129233141819"  />

  - 开机启动 tomcat8

    <img src="img\image-20220129233150281.png" alt="image-20220129233150281"  />

    复制如下脚本

    ```bash
    #!/bin/bash
    #chkconfig:2345 21 90
    #description:apache-tomcat-8
    #processname:apache-tomcat-8
    CATALANA_HOME=/opt/apache-tomcat-8.5.32
    export JAVA_HOME=/opt/java/jdk1.8.0_171
    case $1 in
    start)
        echo "Starting Tomcat..."  
        $CATALANA_HOME/bin/startup.sh
        ;;
    stop)
        echo "Stopping Tomcat..."  
        $CATALANA_HOME/bin/shutdown.sh
        ;;
    
    restart)
        echo "Stopping Tomcat..."  
        $CATALANA_HOME/bin/shutdown.sh
        sleep 2
        echo  
        echo "Starting Tomcat..."  
        $CATALANA_HOME/bin/startup.sh
        ;;
    *)
        echo "Usage: tomcat {start|stop|restart}"  
        ;; esac
    ```

  - 注册服务&添加权限

    <img src="img\image-20220129233243716.png" alt="image-20220129233243716"  />

    <img src="img\image-20220129233246308.png" alt="image-20220129233246308"  />

  - 启动服务&访问 tomcat 测试

    <img src="img\image-20220129233254193.png" alt="image-20220129233254193"  />

    <img src="img\image-20220129233256892.png" alt="image-20220129233256892"  />

- **安装 dubbo-admin**

  dubbo 本身并不是一个服务软件。它其实就是一个 jar 包能够帮你的 java 程序连接到 zookeeper，并利用 zookeeper 消费、提供服务。所以你不用在 Linux 上启动什么 dubbo 服务。

  但是为了让用户更好的管理监控众多的 dubbo 服务，官方提供了一个可视化的监控程序，不过这个监控即使不装也不影响使用。

  - 下载 dubbo-admin

    https://github.com/apache/incubator-dubbo-ops

    <img src="img\image-20220129233405226.png" alt="image-20220129233405226"  />

  - 进入目录，修改 dubbo-admin 配置

    修改 dubbo-admin-server/src/main/resources/application.properties 指定 zookeeper 地址

    <img src="img\image-20220129233416519.png" alt="image-20220129233416519"  />

  - 打包 dubbo-admin

    mvn clean package -Dmaven.test.skip=true

  - 运行 dubbo-admin

    mvn --projects dubbo-admin-server spring-boot:run

    默认使用 root/root 登陆

    <img src="img\image-20220129233444220.png" alt="image-20220129233444220"  />

### 4.dubbo-helloworld

#### 4.1 提出需求

某个电商系统，订单服务需要调用用户服务获取某个用户的所有地址；

我们现在 需要创建两个服务模块进行测试 

| 模块                | 功能           |
| ------------------- | -------------- |
| 订单服务web模块     | 创建订单等     |
| 用户服务service模块 | 查询用户地址等 |

测试预期结果：

​     订单服务web模块在A服务器，用户服务模块在B服务器，A可以远程调用B的功能。

#### 4.2 工程架构

根据 dubbo《服务化最佳实践》

- 分包

  建议将服务接口，服务模型，服务异常等均放在 API 包中，因为服务模型及异常也是 API 的一部分，同时，这样做也符合分包原则：重用发布等价原则(REP)，共同重用原则(CRP)。

  如果需要，也可以考虑在 API 包中放置一份 spring 的引用配置，这样使用方，只需在 spring 加载过程中引用此配置即可，配置建议放在模块的包目录下，以免冲突，如：com/alibaba/china/xxx/dubbo-reference.xml

- 粒度

  服务接口尽可能大粒度，每个服务方法应代表一个功能，而不是某功能的一个步骤，否则将面临分布式事务问题，Dubbo 暂未提供分布式事务支持。

  服务接口建议以业务场景为单位划分，并对相近业务做抽象，防止接口数量爆炸。

  不建议使用过于抽象的通用接口，如：Map query(Map)，这样的接口没有明确语义，会给后期维护带来不便。

<img src="img\image-20220129233620818.png" alt="image-20220129233620818"  />

#### 4.3 创建模块

- gmall-interface：公共接口层（model，service，exception…）

  作用：定义公共接口，也可以导入公共依赖

  - Bean模型

    ```java
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public class UserAddress implements Serializable {
        private Integer id;
        private String userAddress;
        private String userId;
    }
    ```

  - Service接口

    ```java
    public interface UserService {
        List<UserAddress> getUserAddressList(String userId);
    }
    ```

  <img src="img\image-20220129233737114.png" alt="image-20220129233737114"  />

- gmall-user：用户模块（对用户接口的实现）

  - pom.xml

    ```xml
    <dependencies>
        <dependency>
            <groupId>com.atguigu.dubbo</groupId>
            <artifactId>gmall-interface</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
    </dependencies>
    ```

  - Service

    ```java
    public class UserServiceImpl implements UserService {
    
        @Autowired
        UserAddressDao userAddressDao;
    
        @Override
        public List<UserAddress> getUserAddressList(String userId) {
            // TODO Auto-generated method stub
            return userAddressDao.getUserAddressById(userId);
        }
    }
    ```

  - UserAddressDao

    ```java
    @Component
    public class UserAddressDao {
    
        public List<UserAddress> getUserAddressById(String userId) {
            switch (userId){
                case "1":
                    return Collections.singletonList(
                            new UserAddress(1, "wojia1", "100"));
                case "2":
                    return Collections.singletonList(
                            new UserAddress(2, "wojia2", "100"));
                default:
                    return new ArrayList<>();
            }
        }
    }
    ```

- gmall-order-web：订单模块（调用用户模块）

  - pom.xml

    ```xml
    <dependencies>
      	<dependency>
      		<groupId>com.atguigu.dubbo</groupId>
      		<artifactId>gmall-interface</artifactId>
      		<version>0.0.1-SNAPSHOT</version>
      	</dependency>
    </dependencies>
    ```

  - controller

    ```java
    @RestController
    public class OrderController {
    
        @Autowired
        UserService userService;
    
       	@GetMapping("/order")
        public List<UserAddress> getAddress(String userId) {
            return userService.getUserAddressList(userId);
        }
    }
    ```

    现在这样是无法进行调用的。我们 gmall-order-web 引入了 gmall-interface，但是interface的实现是 gmall-user，我们并没有引入，而且实际他可能还在别的服务器中。

#### 4.4 使用 dubbo 改造

> **旧版（了解）**
>
> - 改造 gmall-user 作为服务提供者
>
>   - 引入dubbo
>
>     ```xml
>     <!-- 引入dubbo -->
>     <dependency>
>         <groupId>com.alibaba</groupId>
>         <artifactId>dubbo</artifactId>
>         <version>2.6.2</version>
>     </dependency>
>     <!-- 由于我们使用zookeeper作为注册中心，所以需要操作zookeeper
>     dubbo 2.6以前的版本引入zkclient操作zookeeper 
>     dubbo 2.6及以后的版本引入curator操作zookeeper
>     下面两个zk客户端根据dubbo版本2选1即可
>     -->
>     <dependency>
>         <groupId>com.101tec</groupId>
>         <artifactId>zkclient</artifactId>
>         <version>0.10</version>
>     </dependency>
>     <!-- curator-framework -->
>     <dependency>
>         <groupId>org.apache.curator</groupId>
>         <artifactId>curator-framework</artifactId>
>         <version>2.12.0</version>
>     </dependency>
>     ```
>
>   - 配置提供者
>
>     ```xml
>     <!--当前应用的名字  -->
>     <dubbo:application name="gmall-user"></dubbo:application>
>     <!--指定注册中心的地址  -->
>     <dubbo:registry address="zookeeper://118.24.44.169:2181" />
>     <!--使用dubbo协议，将服务暴露在20880端口  -->
>     <dubbo:protocol name="dubbo" port="20880" />
>     <!-- 指定需要暴露的服务 -->
>     <dubbo:service interface="com.atguigu.gmall.service.UserService" ref="userServiceImpl" />
>     ```
>
>   - 启动服务
>
>     ```java
>     public static void main(String[] args) throws IOException {
>         ClassPathXmlApplicationContext context = 
>             new ClassPathXmlApplicationContext("classpath:spring-beans.xml");
>                   
>         System.in.read(); 
>     }
>     ```
>
> - 改造 gmall-order-web 作为服务消费者
>
>   - 引入dubbo
>
>     ```xml
>     <!-- 引入dubbo -->
>     <dependency>
>         <groupId>com.alibaba</groupId>
>         <artifactId>dubbo</artifactId>
>         <version>2.6.2</version>
>     </dependency>
>     <!-- 由于我们使用zookeeper作为注册中心，所以需要引入zkclient和curator操作zookeeper -->
>     <dependency>
>         <groupId>com.101tec</groupId>
>         <artifactId>zkclient</artifactId>
>         <version>0.10</version>
>     </dependency>
>     <!-- curator-framework -->
>     <dependency>
>         <groupId>org.apache.curator</groupId>
>         <artifactId>curator-framework</artifactId>
>         <version>2.12.0</version>
>     </dependency>
>     ```
>
>   - 配置消费者信息
>
>     ```xml
>     <!-- 应用名 -->
>     <dubbo:application name="gmall-order-web"></dubbo:application>
>     <!-- 指定注册中心地址 -->
>     <dubbo:registry address="zookeeper://118.24.44.169:2181?backup=10.20.153.11:2181,10.20.153.12:2181" />
>     <!-- 生成远程服务代理，可以和本地bean一样使用demoService -->
>     <dubbo:reference id="userService" interface="com.atguigu.gmall.service.UserService"></dubbo:reference>
>     ```
>
> - 测试调用
>
>   访问 gmall-order-web 的 initOrder 请求，会调用 UserService 获取用户地址；
>
>   调用成功。说明我们 order 已经可以调用远程的 UserService 了；

==注解版==

**服务提供方**

- 添加依赖

  ```xml
  <dependency>
      <groupId>org.apache.dubbo</groupId>
      <artifactId>dubbo-spring-boot-starter</artifactId>
      <version>2.7.8</version>
  </dependency>
  <!-- zookeeper 相关导入 -->
  <dependency>
      <groupId>org.apache.curator</groupId>
      <artifactId>curator-framework</artifactId>
      <version>4.3.0</version>
  </dependency>
  <dependency>
      <groupId>org.apache.curator</groupId>
      <artifactId>curator-recipes</artifactId>
      <version>4.3.0</version>
  </dependency>
  ```

- 在 application.properties 添加 dubbo 的相关配置信息，样例配置如下

  ```properties
  spring.application.name=gmall-user
  dubbo.scan.base-packages=com.example.gmalluser.service
  dubbo.protocol.name=dubbo
  dubbo.protocol.port=12345
  dubbo.registry.address=zookeeper://192.168.10.101:2181
  server.port=8081
  ```

- 在 Service 上添加注解

  ```java
  @DubboService(version = "1.0.0")
  public class UserServiceImpl implements UserService {
  
      @Autowired
      UserAddressDao userAddressDao;
  
      @Override
      public List<UserAddress> getUserAddressList(String userId) {
          // TODO Auto-generated method stub
          return userAddressDao.getUserAddressById(userId);
      }
  }
  ```

- 启动你的 Spring Boot 应用，观察控制台，可以看到 dubbo 启动相关信息

**服务消费方**

- 添加依赖

  ```xml
  <dependency>
      <groupId>org.apache.dubbo</groupId>
      <artifactId>dubbo-spring-boot-starter</artifactId>
      <version>2.7.8</version>
  </dependency>
  <dependency>
      <groupId>org.apache.curator</groupId>
      <artifactId>curator-framework</artifactId>
      <version>4.3.0</version>
  </dependency>
  <dependency>
      <groupId>org.apache.curator</groupId>
      <artifactId>curator-recipes</artifactId>
      <version>4.3.0</version>
  </dependency>
  ```

- 在 application.properties 添加 dubbo 的相关配置信息，样例配置如下

  ```properties
  spring.application.name=gmall-order-web
  dubbo.registry.address=zookeeper://192.168.10.101:2181
  server.port=8082
  ```

- <p name="direct">通过 `@DubboReference` 注入 `UserService`</p>

  ```java
  @RestController
  public class OrderController {
  
      // url 表示不通过注册中心直接找到提供者，此时不会注册到 consumer 中
  	// 或者在配置文件中设置 dubbo.registry.address=zookeeper://192.168.10.101:2181
      @DubboReference(version = "1.0.0", url="127.0.0.1:12345", interfaceClass = UserService.class)
      UserService userService;
  
      @GetMapping("/order")
      public List<UserAddress> getAddress(String userId) {
          return userService.getUserAddressList(userId);
      }
  }
  ```

### 5.监控中心

#### 5.1 dubbo-admin

图形化的服务管理页面；安装时需要指定注册中心地址，即可从注册中心中获取到所有的提供者/消费者进行配置管理

#### 5.2 dubbo-monitor-simple

简单的监控中心

- 安装

  - 下载 dubbo-ops

    https://github.com/apache/incubator-dubbo-ops

  - 修改配置指定注册中心地址

    进入 dubbo-monitor-simple\src\main\resources\conf

    修改 dubbo.properties文件

    <img src="img\image-20220129234324152.png" alt="image-20220129234324152"  />

  - 打包dubbo-monitor-simple

    mvn clean package -Dmaven.test.skip=true

  - 解压 tar.gz 文件，并运行start.bat

    <img src="img\image-20220129234339031.png" alt="image-20220129234339031"  />

    如果缺少servlet-api，自行导入 servlet-api 再访问监控中心

  - 启动访问8080

    <img src="img\image-20220129234350049.png" alt="image-20220129234350049"  />

- 监控中心配置

  所有服务配置连接监控中心，进行监控统计

  ```xml
  <!-- 监控中心协议，如果为protocol="registry"，表示从注册中心发现监控中心地址，否则直连监控中心 -->
  <dubbo:monitor protocol="registry"></dubbo:monitor>
  ```

  SpringBoot 中设置：

  ```properties
  dubbo.monitor.protocol=registry
  ```
  
  Simple Monitor 挂掉不会影响到 Consumer 和 Provider 之间的调用，所以用于生产环境不会有风险。
  
  Simple Monitor 采用磁盘存储统计信息，请注意安装机器的磁盘限制，如果要集群，建议用mount共享磁盘。

### 6.整合SpringBoot

- 引入 spring-boot-starter 以及 dubbo 和 curator 的依赖

  ```xml
  <dependency>
      <groupId>com.alibaba.boot</groupId>
      <artifactId>dubbo-spring-boot-starter</artifactId>
      <version>0.2.0</version>
  </dependency>
  ```

  注意starter版本适配：

  <img src="img\image-20220129234459296.png" alt="image-20220129234459296"  />

- 配置 application.properties

  ###### 提供者配置：

  ```properties
  dubbo.application.name=gmall-user
  dubbo.registry.protocol=zookeeper
  dubbo.registry.address=192.168.67.159:2181
  dubbo.scan.base-package=com.atguigu.gmall
  dubbo.protocol.name=dubbo
  ```
  
  application.name就是服务名，不能跟别的dubbo提供端重复
  
  registry.protocol 是指定注册中心协议
  
  registry.address 是注册中心的地址加端口号
  
  protocol.name 是分布式固定是dubbo,不要改。
  
  base-package 注解方式要扫描的包
  
  ###### 消费者配置：
  
  ```properties
  dubbo.application.name=gmall-order-web
  dubbo.registry.protocol=zookeeper
  dubbo.registry.address=192.168.67.159:2181
  dubbo.scan.base-package=com.atguigu.gmall
  dubbo.protocol.name=dubbo
  ```
  
- dubbo 注解

  @Service、@Reference

  【如果没有在配置中写 dubbo.scan.base-package，还需要使用 @EnableDubbo 注解】

---

## 二、dubbo 配置

### 1.配置原则

<img src="img\image-20220129234624431.png" alt="image-20220129234624431"  />

JVM 启动 -D 参数优先，这样可以使用户在部署和启动时进行参数重写，比如在启动时需改变协议的端口。

XML 次之，如果在 XML 中有配置，则 dubbo.properties 中的相应配置项无效。

Properties 最后，相当于缺省值，只有 XML 没有配置时，dubbo.properties 的相应配置项才会生效，通常用于共享公共配置，比如应用名。

duboo.properties 文件会自动加载，不需指定路径，可通过 JVM 启动参数 -Ddubbo.properties.file=xxx.properties 改变缺省配置位置

### 2.启动时检查

在启动时检查依赖的服务是否可用

Dubbo 缺省会在启动时检查依赖的服务是否可用，不可用时会抛出异常，阻止 Spring 初始化完成，以便上线时，能及早发现问题，默认 `check="true"`。

可以通过 `check="false"` 关闭检查，比如，测试时，有些服务不关心，或者出现了循环依赖，必须有一方先启动。

另外，如果你的 Spring 容器是懒加载的，或者通过 API 编程延迟引用服务，请关闭 check，否则服务临时不可用时，会抛出异常，拿到 null 引用，如果 `check="false"`，总是会返回引用，当服务恢复时，能自动连上。

**示例**

- 通过 xml 配置文件

  - 关闭某个服务的启动时检查 (没有提供者时报错)：

    ```xml
    <dubbo:reference interface="com.foo.BarService" check="false" />
    ```

  - 关闭所有服务的启动时检查 (没有提供者时报错)：

    ```xml
    <dubbo:consumer check="false" />
    ```

  - 关闭注册中心启动时检查 (注册中心不存在时是否报错)：

    ```xml
    <dubbo:registry check="false" />
    ```

- 通过 properties 配置文件

  ```properties
  dubbo.reference.com.foo.BarService.check=false
  dubbo.consumer.check=false
  dubbo.registry.check=false
  ```

- 通过注解

  ```java
  @DubboReference(version = "1.0.0", interfaceClass = UserService.class, check = false)
  UserService userService;
  ```

- 通过 -D 参数

  ```sh
  java -Ddubbo.reference.com.foo.BarService.check=false
  java -Ddubbo.consumer.check=false 
  java -Ddubbo.registry.check=false
  ```

**配置的含义**

- `dubbo.reference.com.foo.BarService.check`，覆盖 `com.foo.BarService` 的 reference 的 check 值，就算配置中有声明，也会被覆盖

- `dubbo.consumer.check=false`，是设置 reference 的 `check` 的缺省值，如果配置中有显式的声明，如：`<dubbo:reference check="true"/>`，不会受影响

- `dubbo.registry.check=false`，前面两个都是指订阅成功，但提供者列表是否为空是否报错，如果注册订阅失败时，也允许启动，需使用此选项，将在后台定时重试

### 3.重试次数

失败自动切换，当出现失败，重试其它服务器，但重试会带来更长延迟。可通过 retries="2" 来设置重试次数（不含第一次）

> 幂等操作（多次执行结果相同）可以设置重试次数：查询，删除，修改
>
> 非幂等操作不能设置重试次数：新增

- xml 配置

  ```xml
  重试次数配置如下：
  <dubbo:service retries="2" />
  或
  <dubbo:reference retries="2" />
  或
  <dubbo:reference>
      <dubbo:method name="findFoo" retries="2" />
  </dubbo:reference>
  ```

- SpringBoot 配置（默认 2 次）

  ```properties
  dubbo.consumer.retries=2
  dubbo.provider.retries=2
  ```

  ```java
  @DubboReference(version = "1.0.0", interfaceClass = UserService.class, retries = 2)
  UserService userService;
  ```


### 4.超时时间

由于网络或服务端不可靠，会导致调用出现一种不确定的中间状态（超时）。为了避免超时导致客户端资源（线程）挂起耗尽，必须设置超时时间。

- **Dubbo 消费端**

  - xml 配置

    ```xml
    全局超时配置
    <dubbo:consumer timeout="5000" />
    
    指定接口以及特定方法超时配置，默认 1000
    <dubbo:reference interface="com.foo.BarService" timeout="2000">
        <dubbo:method name="sayHello" timeout="3000" />
    </dubbo:reference>
    ```

  - SpringBoot 配置

    ```properties
    dubbo.consumer.timeout=5000
    ```

    ```java
    @DubboReference(version = "1.0.0", interfaceClass = UserService.class, check = false, timeout = 1000)
    UserService userService;
    ```

- **Dubbo 服务端**

  - xml 配置

    ```xml
    全局超时配置
    <dubbo:provider timeout="5000" />
    
    指定接口以及特定方法超时配置
    <dubbo:provider interface="com.foo.BarService" timeout="2000">
        <dubbo:method name="sayHello" timeout="3000" />
    </dubbo:provider>
    ```

  - SpringBoot 配置

    ```properties
    dubbo.provider.timeout=5000
    ```

    ```java
    @DubboService(version = "1.0.0",timeout = 2000)
    public class UserServiceImpl implements UserService {
    ```

- 配置原则

  dubbo推荐在 Provider 上尽量多配置 Consumer 端属性：

  > 1、作服务的提供者，比服务使用方更清楚服务性能参数，如调用的超时时间，合理的重试次数，等等
  >
  > 2、在Provider配置后，Consumer不配置则会使用Provider的配置值，即Provider配置可以作为Consumer的缺省值。否则，Consumer会使用Consumer端的全局设置，这对于Provider不可控的，并且往往是不合理的

  配置的覆盖规则：

  - 方法级配置别优于接口级别，即小 Scope 优先 
  - Consumer 端配置 优于 Provider 配置 优于 全局配置
  - 最后是 Dubbo Hard Code 的配置值（见配置文档）

  <img src="img\image-20220129234821420.png" alt="image-20220129234821420"  />

### 4.版本号

当一个接口实现，出现不兼容升级时，可以用版本号过渡，版本号不同的服务相互间不引用。

可以按照以下的步骤进行版本迁移：

- 在低压力时间段，先升级一半提供者为新版本
- 再将所有消费者升级为新版本
- 然后将剩下的一半提供者升级为新版本

```xml
老版本服务提供者配置：
<dubbo:service interface="com.foo.BarService" version="1.0.0" />

新版本服务提供者配置：
<dubbo:service interface="com.foo.BarService" version="2.0.0" />

老版本服务消费者配置：
<dubbo:reference id="barService" interface="com.foo.BarService" version="1.0.0" />

新版本服务消费者配置：
<dubbo:reference id="barService" interface="com.foo.BarService" version="2.0.0" />

如果不需要区分版本，可以按照以下的方式配置（随即调用版本）：
<dubbo:reference id="barService" interface="com.foo.BarService" version="*" />
```

### 5.本地存根

在 consumer 或 provider 中创建 UserServiceStub 类，编写参数的验证逻辑，若验证通过，则调用远程代理对象

**Stub 必须有可传入 Proxy 的构造函数**

```java
public class UserServiceStub implements UserService {

    private final UserService userService;

    /**
     * 传入的是 UserService 的远程代理对象
     * @param userService
     */
    public UserServiceStub(UserService userService) {
        this.userService = userService;
    }

    @Override
    public List<UserAddress> getUserAddressList(String userId) {
        // 若参数值为 1 则调用
        if (Integer.parseInt(userId) == 1) {
            return userService.getUserAddressList(userId);
        }
        return null;
    }
}
```

- xml 配置方式

  ```xml
  <dubbo:consumer interface="com.foo.BarService" stub="true" />
  或
  <dubbo:consumer interface="com.foo.BarService" stub="com.foo.BarServiceStub" />
  ```

- SpringBoot 配置方式

  ```java
  @DubboReference(version = "1.0.0", interfaceClass = UserService.class, stub="com.example.gmallorderweb.service.UserServiceStub")
  UserService userService;
  ```

### ==6.与 SpringBoot 整合的三种方式==

1. 导入 dubbo-starter

   - 在 application.properties 配置属性，使用 @DubboService【暴露服务】和 @DubboReference【引用服务】
   - 在服务提供方添加包扫描：`dubbo.scan.base-packages` 或添加 `@EnableDubbo` 注解

2. 保留 dubbo xml 配置文件

   导入 starter，仍然使用 xml 方式时，在主程序上添加：

   ```java
   @ImportResource(locations="classpath:provider.xml")
   ```

3. 使用配置类

   将每个组件手动创建到容器中

   例：provider

   - config 文件

     ```java
     @Configuration
     @EnableDubbo(scanBasePackages = "com.example.gmallorderweb")
     public class MyDubboConfig {
     
         @Bean
         public ApplicationConfig applicationConfig() {
             ApplicationConfig applicationConfig = new ApplicationConfig();
             applicationConfig.setName("gmnall-order-web");
             return applicationConfig;
         }
     
         @Bean
         public RegistryConfig registryConfig() {
             RegistryConfig registryConfig = new RegistryConfig();
             registryConfig.setProtocol("zookeeper");
             registryConfig.setAddress("192.168.10.101:2181");
             return registryConfig;
         }
     
         @Bean
         public ProtocolConfig protocolConfig() {
             ProtocolConfig protocolConfig = new ProtocolConfig();
             protocolConfig.setName("dubbo");
             protocolConfig.setPort(12345);
             return protocolConfig;
         }
     
         @Bean
         public ServiceConfig<UserService> serviceConfig(UserService userService) {
             ServiceConfig<UserService> serviceConfig = new ServiceConfig<>();
             serviceConfig.setInterface(UserService.class);
             serviceConfig.setRef(userService);
             serviceConfig.setVersion("1.0.0");
     
             MethodConfig methodConfig = new MethodConfig();
             methodConfig.setName("getUserAddressList");
             methodConfig.setTimeout(1000);
     
             //ConsumerConfig
             //MonitorConfig
     
             serviceConfig.setMethods(Collections.singletonList(methodConfig));
             return serviceConfig;
         }
     }
     ```

   - application.properties

     ```properties
     server.port=8081
     ```

   - UserServiceImpl

     ```java
     @Component
     public class UserServiceImpl implements UserService {
     
         @Autowired
         UserAddressDao userAddressDao;
     
         @Override
         public List<UserAddress> getUserAddressList(String userId) {
             // TODO Auto-generated method stub
             return userAddressDao.getUserAddressById(userId);
         }
     }
     ```

     

---

## 三、高可用

### 1.zookeeper 宕机与 dubbo 直连

现象：zookeeper 注册中心宕机，还可以消费 dubbo 暴露的服务。

原因：

> 健壮性
>
> - 监控中心宕掉不影响使用，只是丢失部分采样数据
> - 数据库宕掉后，注册中心仍能通过缓存提供服务列表查询，但不能注册新服务
> - 注册中心对等集群，任意一台宕掉后，将自动切换到另一台
> - **注册中心全部宕掉后，服务提供者和服务消费者仍能通过本地缓存通讯**
> - 服务提供者无状态，任意一台宕掉后，不影响使用
> - 服务提供者全部宕掉后，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复

高可用：通过设计，减少系统不能提供服务的时间；

直连方式：直接通过 dubbo 直连服务提供者，[使用 url 属性](#direct)

### 2.集群下 dubbo 负载均衡配置

在集群负载均衡时，Dubbo 提供了多种均衡策略，缺省为 random 随机调用。

**负载均衡策略**

> **Random LoadBalance**
>
> - **加权随机**，按权重设置随机概率。
> - 在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重。
> - 缺点：存在慢的提供者累积请求的问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上。
>
> <img src="img\image-20220131200126709.png" alt="image-20220131200126709" style="zoom: 50%;" />
>
> **RoundRobin LoadBalance**
>
> - **加权轮询**，按公约后的权重设置轮询比率，循环调用节点
> - 缺点：同样存在慢的提供者累积请求的问题。
>
> 加权轮询过程过程中，如果某节点权重过大，会存在某段时间内调用过于集中的问题。
>
> 存在慢的提供者累积请求的问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上。
>
> <img src="img\image-20220131200142457.png" alt="image-20220131200142457" style="zoom: 50%;" />
>
> **LeastActive LoadBalance**
>
> - **加权最少活跃调用优先**，活跃数越低，越优先调用，相同活跃数的进行加权随机。活跃数指调用前后计数差（针对特定提供者：请求发送数 - 响应返回数），表示特定提供者的任务堆积量，活跃数越低，代表该提供者处理能力越强。
> - 使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大；相对的，处理能力越强的节点，处理更多的请求。
>
> <img src="img\image-20220131200154590.png" alt="image-20220131200154590" style="zoom:50%;" />
>
> ### ShortestResponse
>
> - **加权最短响应优先**，在最近一个滑动窗口中，响应时间越短，越优先调用。相同响应时间的进行加权随机。
> - 使得响应时间越快的提供者，处理更多的请求。
> - 缺点：可能会造成流量过于集中于高性能节点的问题。
>
> 这里的响应时间 = 某个提供者在窗口时间内的平均响应时间，窗口时间默认是 30s。
>
> **ConsistentHash LoadBalance**
>
> - **一致性 Hash**，相同参数的请求总是发到同一提供者。
> - 当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。
> - 算法参见：[Consistent Hashing | WIKIPEDIA](http://en.wikipedia.org/wiki/Consistent_hashing)
> - 缺省只对第一个参数 Hash，如果要修改，请配置 `<dubbo:parameter key="hash.arguments" value="0,1" />`
> - 缺省用 160 份虚拟节点，如果要修改，请配置 `<dubbo:parameter key="hash.nodes" value="320" />`
>
> <img src="img\image-20220131200208768.png" alt="image-20220131200208768" style="zoom: 50%;" />

**配置**

服务端服务级别

```xml
<dubbo:service interface="..." loadbalance="roundrobin" />
```

客户端服务级别

```xml
<dubbo:reference interface="..." loadbalance="roundrobin" />
```

服务端方法级别

```xml
<dubbo:service interface="...">
    <dubbo:method name="..." loadbalance="roundrobin"/>
</dubbo:service>
```

客户端方法级别

```xml
<dubbo:reference interface="...">
    <dubbo:method name="..." loadbalance="roundrobin"/>
</dubbo:reference>
```

SpringBoot 方法

```properties
dubbo.consumer.loadbalance=
dubbo.provider.loadbalance=
```

```java
@DubboReference(version = "1.0.0", interfaceClass = UserService.class, loadbalance = )
```

```java
@DubboService(loadbalance = )
```

```java
methodConfig.setLoadbalance();
```

> 权重可以在服务提供者那里调节，也可以在 Dubbo admin 控制台动态调节

### 3.整合 hystrix，服务熔断与降级处理

- 服务降级

  > **什么是服务降级？**
  >
  > 当服务器压力剧增的情况下，根据实际业务情况及流量，对一些服务和页面有策略的不处理或换种简单的方式处理，从而释放服务器资源以保证核心交易正常运作或高效运作。

  可以通过服务降级功能临时屏蔽某个出错的非关键服务，并定义降级后的返回策略。

  向注册中心写入动态配置覆盖规则：

  ```java
  RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getAdaptiveExtension();
  Registry registry = registryFactory.getRegistry(URL.valueOf("zookeeper://10.20.153.10:2181"));
  registry.register(URL.valueOf("override://0.0.0.0/com.foo.BarService?category=configurators&dynamic=false&application=foo&mock=force:return+null"));
  ```

  其中：

  - mock=force:return+null 表示消费方对该服务的方法调用都直接返回 null 值，不发起远程调用。用来屏蔽不重要服务不可用时对调用方的影响

    可在 dubbo 控制中心直接禁用某服务

    <img src="img\image-20220131231004783.png" alt="image-20220131231004783" style="zoom:80%;" />

  - 还可以改为 mock=fail:return+null 表示消费方对该服务的方法调用在失败后，再返回 null 值，不抛异常。用来容忍不重要服务不稳定时对调用方的影响

    可在 dubbo 控制中心直接容错某服务

- 集群容错

  在集群调用失败时，Dubbo 提供了多种容错方案，缺省为 failover 重试

  **集群容错模式**

  > **Failover Cluster**
  > 失败自动切换，当出现失败，重试其它服务器。通常用于读操作，但重试会带来更长延迟。可通过 retries="2" 来设置重试次数(不含第一次)。
  >
  > ```xml
  > 重试次数配置如下：
  > <dubbo:service retries="2" />
  > 或
  > <dubbo:reference retries="2" />
  > 或
  > <dubbo:reference>
  >     <dubbo:method name="findFoo" retries="2" />
  > </dubbo:reference>
  > ```
  >
  > **Failfast Cluster**
  > 快速失败，只发起一次调用，失败立即报错。通常用于非幂等性的写操作，比如新增记录。
  >
  > **Failsafe Cluster**
  > 失败安全，出现异常时，直接忽略。通常用于写入审计日志等操作。
  >
  > **Failback Cluster**
  > 失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作。
  >
  > **Forking Cluster**
  > 并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过 forks="2" 来设置最大并行数。
  >
  > **Broadcast Cluster**
  > 广播调用所有提供者，逐个调用，任意一台报错则报错 [2]。通常用于通知所有提供者更新缓存或日志等本地资源信息。
  >
  > 集群模式配置
  > 按照以下示例在服务提供方和消费方配置集群模式
  >
  > ```xml
  > <dubbo:service cluster="failsafe" />
  > 或
  > <dubbo:reference cluster="failsafe" />
  > ```

- 整合 hystrix

  > ​    Hystrix 旨在通过控制那些访问远程系统、服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。Hystrix 具备拥有回退机制和断路器功能的线程和信号隔离，请求缓存和请求打包，以及监控和配置等功能。

  - 配置 spring-cloud-starter-netflix-hystrix

    spring boot 官方提供了对 hystrix 的集成，直接在 pom.xml 里加入依赖：

    ```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        <version>1.4.4.RELEASE</version>
    </dependency>
    ```

    然后在 Application 类上增加 @EnableHystrix 来启用 hystrix starter：

    ```java
    @SpringBootApplication
    @EnableHystrix
    public class ProviderApplication {
    ```

  - 配置 Provider 端

    在 Dubbo的Provider 上增加 @HystrixCommand 配置，这样子调用就会经过 Hystrix 代理。

    ```java
    @Service(version = "1.0.0")
    public class HelloServiceImpl implements HelloService {
        @HystrixCommand(commandProperties = {
            
        	@HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),
            // 设置调用者执行的超时时间（单位毫秒）
        	@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "2000") 
        }
                       )
        @Override
        public String sayHello(String name) {
            // System.out.println("async provider received: " + name);
            // return "annotation: hello, " + name;
            throw new RuntimeException("Exception to show hystrix enabled.");
        }
    }
    ```

  - 配置 Consumer 端
  
    对于 Consumer 端，则可以增加一层 method 调用，并在 method 上配置 @HystrixCommand。当调用出错时，会走到 fallbackMethod = "reliable" 的调用里。
  
    ```java
    @Reference(version = "1.0.0")
    private HelloService demoService;
    
    // 出错时调用 reliable 方法
    @HystrixCommand(fallbackMethod = "reliable")
    public String doSayHello(String name) {
        return demoService.sayHello(name);
    }
    public String reliable(String name) {
        return "hystrix fallback value";
    }
    ```

---

## 四、dubbo 原理

### 1.RPC 原理

<img src="img\image-20220129235414821.png" alt="image-20220129235414821"  />

一次完整的 RPC 调用流程（同步调用，异步另说）如下：

- **服务消费方（client）调用以本地调用方式调用服务**

- client stub 接收到调用后负责将接口名、方法、参数、返回值、`version`（用来确定调用哪一个实现类）等组装成能够进行网络传输的消息体

- client stub 从注册中心找到服务端地址，并将消息发送到服务端

  > **提示**
  >
  > 在 ZooKeeper 上，Dubbo 通常会创建一个根节点，例如 `/dubbo`，然后在该节点下创建服务接口的节点，每个节点包含了提供者和消费者的信息。
  >
  > 例如：
  >
  > ```bash
  > bashCopy code/dubbo
  > └── com.example.api.SomeService
  >     ├── providers
  >     │   └── provider-1.0.0
  >     │       └── 192.168.1.100:20880
  >     └── consumers
  >         └── consumer-1.0.0
  >             └── 192.168.1.101:20881
  > ```
  >
  > 其中，`com.example.api.SomeService` 是服务接口的全限定名，`provider-1.0.0` 和 `consumer-1.0.0` 是服务的版本号，`192.168.1.100:20880` 和 `192.168.1.101:20881` 是提供者和消费者的地址。

- server stub 收到消息后进行解码

- server stub 根据解码结果反射生成相应的 class，进而调用相应的方法

- 本地服务执行并将结果返回给 server stub

- server stub 注册中心找到消费端地址，将返回结果打包成消息并发送至消费方

- client stub 接收到消息，并进行解码

- **服务消费方得到最终结果**

RPC框架的目标就是要2~8这些步骤都封装起来，这些细节对用户来说是透明的，不可见的。

### 2.netty 通信原理

Netty 是一个异步事件驱动的网络应用程序框架， 用于快速开发可维护的高性能协议服务器和客户端。它极大地简化并简化了 TCP 和 UDP 套接字服务器等网络编程。

**BIO：(Blocking IO)**

<img src="img\image-20220129235545365.png" alt="image-20220129235545365"  />

**NIO (Non-Blocking IO)**

<img src="img\image-20220129235551768.png" alt="image-20220129235551768"  />

Selector 一般称 为**选择器** ，也可以翻译为 **多路复用器，**

Connect（连接就绪）、Accept（接受就绪）、Read（读就绪）、Write（写就绪）

Netty基本原理：

<img src="img\image-20220129235603395.png" alt="image-20220129235603395" style="zoom:150%;" />

### 3.dubbo 原理

> 暴露服务和引用时底层开启 netty 服务器

- **dubbo原理 - 框架设计**

  <img src="img\image-20220131234052120.png" alt="image-20220131234052120" style="zoom: 150%;" />

  - config 配置层：对外配置接口，以 ServiceConfig, ReferenceConfig 为中心，可以直接初始化配置类，也可以通过 spring 解析配置生成配置类
  - proxy 服务代理层：服务接口透明代理，生成服务的客户端 Stub 和服务器端 Skeleton, 以 ServiceProxy 为中心，扩展接口为 ProxyFactory
  - registry 注册中心层：封装服务地址的注册与发现，以服务 URL 为中心，扩展接口为 RegistryFactory, Registry, RegistryService
  - cluster 路由层：封装多个提供者的路由及负载均衡，并桥接注册中心，以 Invoker 为中心，扩展接口为 Cluster, Directory, Router, LoadBalance
  - monitor 监控层：RPC 调用次数和调用时间监控，以 Statistics 为中心，扩展接口为 MonitorFactory, Monitor, MonitorService
  - protocol 远程调用层：封装 RPC 调用，以 Invocation, Result 为中心，扩展接口为 Protocol, Invoker, Exporter
  - exchange 信息交换层：封装请求响应模式，同步转异步，以 Request, Response 为中心，扩展接口为 Exchanger, ExchangeChannel, ExchangeClient, ExchangeServer
  - transport 网络传输层：抽象 mina 和 netty 为统一接口，以 Message 为中心，扩展接口为 Channel, Transporter, Client, Server, Codec
  - serialize 数据序列化层：可复用的一些工具，扩展接口为 Serialization, ObjectInput, ObjectOutput, ThreadPool

- **dubbo原理 - 启动解析、加载配置信息**

  <img src="img\image-20220129235709351.png" alt="image-20220129235709351"  />

- **dubbo原理 - 服务暴露**

  <img src="img\image-20220131234136030.png" alt="image-20220131234136030"  />

- **dubbo原理 - 服务引用**

  <img src="img\image-20220131234154784.png" alt="image-20220131234154784" style="zoom:80%;" />

- **dubbo原理 - 服务调用**

  <img src="img\image-20220129235748245.png" alt="image-20220129235748245" style="zoom:150%;" />

  