# 第1章_SpringCloud简介

## 1.简介

微服务架构提倡将单一应用程序划分成一组小的服务，每个服务运行在独立的进程中，服务与服务间采用轻量级的通信机制互相协作（例如基于 HTTP 协议的 RESTful API）。

SpringCloud 是分布式微服务架构的一站式解决方案，是多种微服务架构落地技术的集合体，俗称微服务全家桶。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/20200603155055794-fc07cd451761e5e06eb2c996682d5ef7-eec8bd.png" alt="img" style="zoom: 33%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/2020060315584195-ea7d2e5f5061ee36b08eb3cac7b6be3d-cc347c.png" alt="在这里插入图片描述" style="zoom: 33%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/20200603160047209-8a41543f45157695aba72b6e87d9519b-12c8c1.png" alt="在这里插入图片描述" style="zoom:33%;" />

**springcloud与springboot版本的对应关系**

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