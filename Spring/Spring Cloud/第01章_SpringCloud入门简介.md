# 第01章_SpringCloud入门简介

## 1.Spring Cloud简介

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

## 2.Spring Cloud Alibaba简介

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

**Pom 示例**

可以使用阿里的云原生脚手架：https://start.aliyun.com

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
    <name>hello-world</name>
    <packaging>pom</packaging>
    <description>hello world project project for Spring Cloud Alibaba</description>

    <properties>
        <java.version>1.8</java.version>
        <spring-cloud-alibaba.version>2021.0.4.0</spring-cloud-alibaba.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
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
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

