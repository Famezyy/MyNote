# 第04章_Docker进阶

## 1.Docker file

Dockerfile 是用来构建 Docker 镜像的文本文件，是由一条条构建镜像所需的指令和参数构成的脚本。

官网：https://docs.docker.com/engine/reference/builder/

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230122220731223-4142ba0c93cebb98a94613ccfb4a8be1-c32c54.png" alt="image-20230122220731223" style="zoom:67%;" />

**构建三步骤**

- 编写 Dockerfile 文件
- `docker build`命令构建镜像
- `docker run`依镜像运行容器实例

### 1.1 构建过程解析

**基础知识**

- 每条保留字指令都必须为大写字母且后面要跟随至少一个参数
- 指令按照从上到下，顺序执行
- #表示注释
- 每条指令都会创建一个新的镜像层并对镜像进行提交

**Docker 执行 Dockerfile 的大致流程**

- docker 从基础镜像运行一个容器
- 执行一条指令并对容器作出修改
- 执行类似 docker commit 的操作提交一个新的镜像层
- docker 再基于刚提交的镜像运行一个新容器
- 执行 dockerfile 中的下一条指令直到所有指令都执行完成

从应用软件的角度来看，Dockerfile、Docker 镜像与 Docker 容器分别代表软件的三个不同阶段

- Dockerfile 是软件的原材料，涉及的内容包括执行代码或者是文件、环境变量、依赖包、运行时环境、动态链接库、操作系统的发行版、服务进程和内核进程(当应用进程需要和系统服务和内核进程打交道，这时需要考虑如何设计 namespace 的权限控制)等等

- Docker 镜像是软件的交付品，在用 Dockerfile 定义一个文件之后，`docker build`时会产生一个 Docker 镜像，当运行 Docker 镜像时会真正开始提供服务

- Docker 容器则可以认为是软件镜像的运行态，也即依照镜像运行的容器实例

Dockerfile 面向开发，Docker 镜像成为交付标准，Docker 容器则涉及部署与运维，三者缺一不可，合力充当 Docker 体系的基石。

### 1.2 常用保留字指令

参考 tomcat8 的 dockerfile 入门：https://github.com/docker-library/tomcat

- `FROM`：基础镜像，当前新镜像是基于哪个镜像的，指定一个已经存在的镜像作为模板，第一条必须是 from

- `MAINTAINER`：镜像维护者的姓名和邮箱地址

- `RUN`：容器构建时需要运行的命令，在`docker build`时运行

  - shell 格式

    ```bash
    RUN <命令行命令>
    # 例
    RUN yum install -y net-tool
    ```

  - exec 格式

    ```bash
    RUN ["可执行文件", "参数1", "参数2"]
    # 例
    RUN ["/bin/bash","./start.sh"]
    ```

- `EXPOSE`：当前容器对外暴露出的端口，可以一次设置多个端口号，用空格隔开

  或者可以在`docker run`时指定`--expose=1234`，这两种方式作用相同。但是，`--expose`可以接受端口范围作为参数，比如`--expose=2000-3000`。`EXPOSE`和`--expose`都不依赖于宿主机器。默认状态下，这些规则并不会使这些端口可以通过宿主机来访问。

  `EXPOSE`指令是声明容器运行时提供服务的端口，这**只是一个声明**，在容器运行时并不会因为这个声明应用就会开启这个端口的服务。在 Dockerfile 中写入这样的声明有两个好处，一个是帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射；另一个用处则是在运行时使用随机端口映射时，也就是`docker run -P`时，会自动随机映射`EXPOSE`的端口。

  要将`EXPOSE`和在运行时使用`-p <宿主端口>:<容器端口>`区分开来。`-p`，是映射宿主端口和容器端口，换句话说，就是将容器的对应端口服务公开给外界访问，而`EXPOSE`仅仅是声明容器打算使用什么端口而已，并不会自动在宿主进行端口映射。
  
- `WORKDIR`：指定在创建容器后，终端默认登陆的进来工作目录，一个落脚点

- `USER`：指定该镜像以什么样的用户去执行，如果都不指定，默认是 root

- `ENV`：用来在构建镜像过程中设置环境变量

  这个环境变量可以在后续的任何 RUN 指令中使用，也可以在其它指令中直接使用这些环境变量，比如：

  ```bash
  ENV MY_PATH /usr/mytest
  WORKDIR $MY_PATH
  ```

- `ADD`：将宿主机上下文目录下的文件拷贝进镜像且会自动处理 URL 和解压 tar 压缩包

- `COPY`：类似`ADD`，拷贝文件和目录到镜像中。 将从构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置

  ```bash
  COPY src dest
  COPY ["src", "dest"]
  <src源路径>：源文件或者源目录
  <dest目标路径>：容器内的指定路径，该路径不用事先建好，路径不存在的话，会自动创建
  ```

- `VOLUME`：容器数据卷，用于数据保存和持久化工作，无法指定主机上对应的目录，默认会在主机上的`/var/lib/docker/volumes/`下创建文件夹用于挂载容器中的目录

  例如以下命令会将容器的`/usr/local/file`挂载到主机上的`/var/lib/docker/volumes/随机的文件名/`文件夹下

  ```bash
  VOLUME /usr/local/file/
  ```

  `VOLUME`只是指定了一个目录，用以在用户忘记启动时指定`-v`参数也可以保证容器的正常运行。比如 mysql，你不能说用户启动时没有指定`-v`，然后删了容器，就把 mysql 的数据文件都删了，那样生产上是会出大事故的，所以 mysql 的 Dockerfile 里面就需要配置`VOLUME`，这样即使用户没有指定`-v`，容器被删后也不会导致数据文件都不在了。还是可以恢复的。

  如果`-v`和`volume`指定了不同的位置，会以`-v`设定的目录为准，其实`volume`指令的设定的目的就是为了避免用户忘记指定`-v`的时候导致的数据丢失，那么如果用户指定了`-v`，自然而然就不需要`VOLUME`指定的位置了。

- `CMD`：指定容器启动后的要干的事情

  - shell 格式：`CMD<命令>`

  - exec 格式：`CMD["可执行文件", "参数1", "参数2"]`

  - 参数列表格式：`CMD["参数1", "参数2"]`，用于在指定了`ENTRYPOINT`后，用`CMD`制定具体的参数

    > **注意**
    >
    > Dockerfile 中可以有多个`CMD`指令，但挂起命令只有最后一个生效，如果执行的`docker run`命令带参数，则`CMD`指令无效。
    >
    > 参考官网 Tomcat 的 dockerfile 演示讲解，官网最后一行命令：`CMD ["catalina.sh", "run"]`
    >
    > 直接运行`docker run -it -p 8080:8080 tomcat`会启动 tomcat 服务器，但是运行`docker run -it -p 8080:8080 tomcat /bin/bash`则会打开`bash`窗口，不会启动服务器

- `ENTRYPOINT`：用来指定一个容器启动时要运行的命令

  类似于`CMD`指令，但是一般`ENTRYPOINT`不会被`docker run`后面的命令覆盖， 而且这些命令行参数会被当作参数送给`ENTRYPOINT`指令指定的程序。

  命令格式：`ENTRYPOINT ["可执行文件", "参数1", "参数2"]`。

  > 要想使`ENTRYPOINT`命令参数可被`docker run`命令行参数中指定要运行的命令覆盖，需要使用`--entrypoint`选项进行显式覆盖：
  >
  > ```bash
  > docker run --name demo3C --rm -it --entrypoint ifconfig demo3:test
  > ```
  >
  > 当然使用`--entrypoint`选项进行显式覆盖命令时，依然可以传递参数
  >
  > ```bash
  > docker run --name demo5D --rm -it --entrypoint ping demo5:test weibo.com
  > ```

  `ENTRYPOINT`可以和`CMD`一起用，这里的`CMD`等于是在给`ENTRYPOINT`传参。当指定了`ENTRYPOINT`后，`CMD`的含义就发生了变化，不再是直接运行其命令而是将`CMD`的内容作为参数传递给`ENTRYPOINT`指令。

  案例如下，假设已通过 Dockerfile 构建了 nginx:test 镜像：

  ```bash
  FROM nginx
  ENTRYPOINT ["nginx", "-c"]
  CMD ["/etc/nginx/nginx.conf"]
  ```

  |     是否传参     |     按照 dockerfile 编写执行     |                  传参运行                   |
  | :--------------: | :------------------------------: | :-----------------------------------------: |
  |   Docker 命令    |     `docker run nginx:test`      | `docker run nginx:test /etc/nginx/new.conf` |
  | 衍生出的实际命令 | `nginx -c /etc/nginx/nginx.conf` |       `nginx -c /etc/nginx/new.conf`        |

### 1.3 案例1

自定义 Centos7 镜像具备 vim+ifconfig+jdk8

JDK 的下载镜像地址：https://www.oracle.com/java/technologies/downloads/#java8

或者：https://mirrors.yangxingzhen.com/jdk/

**编写 Dockerfile 文件**

```bash
$ vim Dockerfile
```

```bash
FROM centos:centor7
MAINTAINER zzyy<zzyybs@126.com>
 
ENV MYPATH /usr/local
WORKDIR $MYPATH
 
#安装vim编辑器
RUN yum -y install vim
#安装ifconfig命令查看网络IP
RUN yum -y install net-tools
#安装java8及lib库
RUN yum -y install glibc.i686
RUN mkdir /usr/local/java
#ADD 把同目录下 jdk-8u171-linux-x64.tar.gz 添加到容器中,安装包必须要和 Dockerfile 文件在同一位置
ADD jdk-8u171-linux-x64.tar.gz /usr/local/java/
#配置 java 环境变量
ENV JAVA_HOME /usr/local/java/jdk1.8.0_171
ENV JRE_HOME $JAVA_HOME/jre
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib:$CLASSPATH
ENV PATH $JAVA_HOME/bin:$PATH
 
EXPOSE 80
CMD echo $MYPATH
CMD echo "success--------------ok"
CMD /bin/bash
```

**构建**

```bash
$ docker build -t 新镜像名字:TAG .
$ docker build -t centosjava8:1.5 .
```

**运行**

```bash
$ docker run -it 新镜像名字:TAG
$ docker run -it centosjava8:1.5 /bin/bash
```

### 1.4 案例2

自定义镜像 ubuntu

**编写 DockerFile 文件**

```bash
FROM ubuntu
MAINTAINER zzyy<zzyybs@126.com>
 
ENV MYPATH /usr/local
WORKDIR $MYPATH

# 更换为 ali 源
RUN sed -i s/archive.ubuntu.com/mirrors.aliyun.com/g /etc/apt/sources.list \
  && sed -i s/security.ubuntu.com/mirrors.aliyun.com/g /etc/apt/sources.list \
  && apt-get update && apt-get upgrade
 
RUN apt-get update
RUN apt-get install net-tools
#RUN apt-get install -y iproute2
#RUN apt-get install -y inetutils-ping
 
EXPOSE 80
 
CMD echo $MYPATH
CMD echo "install inconfig cmd into ubuntu success--------------ok"
CMD /bin/bash
```

**构建**

```bash
$ docker build -t 新镜像名字:TAG .
```

**运行**

```bash
$ docker run -it 新镜像名字:TAG
```

> **更换 centos8 镜像的源**
>
> ```bash
> # 进入到配置文件
> cd /etc/yum.repos.d/
> vi aliyun.repo
> # 将一下代码写进aliyun.repo里面
> cat aliyun.repo
> [AppStream]
> name=AppStream
> baseurl=https://mirrors.aliyun.com/centos/8.5.2111/AppStream/x86_64/os/
> enabled=1
> gpgcheck=0
> 
> [BaseOS]
> name=BaseOS
> baseurl=https://mirrors.aliyun.com/centos/8.5.2111/BaseOS/x86_64/os/
> enabled=1
> gpgcheck=0
> 
> # 删除原有的数据缓存并重新建立数据缓存
> yum clean all && yum makecache fast
> yum repolist
> ```

### 1.5 删除虚悬镜像

虚悬镜像是仓库名、标签都是\<none\>的镜像，俗称dangling image。如果构建失败会产生虚悬镜像。

**查看虚悬镜像**

```bash
$ docker images -f dangling=true
```

**删除**

```bash
$ docker image prune
```

## 2.微服务实战

- 通过 IDEA 新建一个普通微服务模块并打包成一个 jar 包

- 通过 Dockerfile 发布微服务部署到 docker 容器

  - 编写 Dockerfile

    ```bash
    # 基础镜像使用java
    FROM openjdk:8
    # 作者
    MAINTAINER youyi
    # VOLUME 指定临时文件目录为/tmp，在主机 /var/lib/docker 目录下创建了一个临时文件夹并链接到容器的 /tmp
    VOLUME /tmp
    # 将 jar 包添加到容器中并更名为 microservice.jar
    ADD docker_boot-0.0.1-SNAPSHOT.jar /microservice.jar
    # 运行jar包
    ENTRYPOINT ["java","-jar","/microservice.jar"]
    #暴露 8080 端口作为微服务，要和微服务的端口一致
    EXPOSE 8080
    ```

  - 将微服务 jar 包和 Dockerfile 文件上传到同一个目录下

  - 执行`docker build -t microservice:1.0 .`构建容器

  - 执行`docker run -d -p 6001:6001 microservice:1.0`运行容器

## 3.Docker网络

docker 不启动时，默认有三个网络参数：`ens33`、`lo`、`virbr0`。

在 CentOS7 的安装过程中如果有选择相关虚拟化的的服务安装系统后，启动网卡时会发现有一个以网桥连接的私网地址的 virbr0 网卡（virbr0 网卡：它还有一个固定的默认IP地址 192.168.122.1），是做虚拟机网桥的使用的，其作用是为连接其上的虚机网卡提供 NAT 访问外网的功能。

我们之前学习 Linux 安装，勾选安装系统的时候附带了 libvirt 服务才会生成的一个东西，如果不需要可以直接将 libvirtd 服务卸载

```bash
$ yum remove libvirt-libs.x86_64
```

docker 启动后，会产生一个名为 docker0 的虚拟网桥：`docker0`、`ens33`、`lo`

查看 docker 网络：

```bash
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
aea5aa18dd94   bridge    bridge    local
d57374a995d7   host      host      local
c03ed094b709   none      null      local
```

### 3.1 常用命令

- 查询网络：`docker network ls`
- 创建网络：`docker network create [网络名字]`
- 查看网络源数据：`docker network inspect [网络名字]`
- 删除网络：`docker network rm [网络名字]`

### 3.2 网络模式

| 网络模式  | 简介                                                         |
| --------- | ------------------------------------------------------------ |
| bridge    | `--network bridge`，为每一个容器分配、设置 IP 等，并将容器连接到一个`docker0`虚拟网桥，默认为`bridge`模式 |
| host      | `--network host`，容器不会虚拟出自己的网卡，配置自己的 IP 等，而是使用宿主机的 IP 和端口 |
| none      | `--network none`，容器有独立的 Network namespace，但并没有对其进行任何网络设置，如分配 veth pair 和网桥连接，IP 等 |
| container | `--network container:NAME`，新创建的容器不会创建自己的网卡和配置自己的 IP，而是和一个指定的容器共享 IP、端口范围等 |

**容器实例内默认网络 IP 生产规则**

先启动两个 ubuntu 容器

```bash
$ docker run -it --name u1 ubuntu bash
$ docker run -it --name u2 ubuntu bash
```

检查容器可以发现，u1 分配的 IP 为 172.17.0.2， u2 分配的 IP 为 172.17.0.3。

```bash
$ docker inspect u1 | tail -n 20
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "aea5aa18dd946a6671f21669249c154a5ef012f610e975a1d5565f3459ac7573",
                    "EndpointID": "28518501381cdfa9dee850fe50cd47977e75bbe431255b3b3aeed4f6e5034621",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:02",
                    "DriverOpts": null
                }
            }
        }
    }
]
```

```bash
$ docker inspect u2 | tail -n 20
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "aea5aa18dd946a6671f21669249c154a5ef012f610e975a1d5565f3459ac7573",
                    "EndpointID": "003069fc0f846ea4b0ff33e900580ee576669322b68a3ece6f8f55266ffbd0aa",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.3",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:03",
                    "DriverOpts": null
                }
            }
        }
    }
]
```

接下来关闭 u2，新建 u3，查看 IP 变化

```bash
$ docker stop u2
$ docker run -it --name u3 ubuntu bash
$ docker inspect u3 | tail -n 20
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "aea5aa18dd946a6671f21669249c154a5ef012f610e975a1d5565f3459ac7573",
                    "EndpointID": "c2167be3811cb2a4c6481f4a7d8ccc4221628739d3998fee46504488a998829a",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.3",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:03",
                    "DriverOpts": null
                }
            }
        }
    }
]
```

可以发现，u3 分配的 IP 为 172.17.0.3。说明 docker 容器内部的 IP 是有可能发生变换的。

#### 1.bridge

Docker 服务默认会创建一个 docker0 网桥（其上有一个 docker0 内部接口），该桥接网络的名称为 docker0，它在内核层连通了其他的物理或虚拟网卡，将所有容器和本地主机都放到同一个物理网络。Docker 默认指定了 docker0 接口 的 IP 地址和子网掩码，让主机和容器之间可以通过网桥相互通信。

查看 bridge 网络的详细信息，并通过 grep 获取名称项：

```bash
$ docker network inspect bridge | grep name
            "com.docker.network.bridge.name": "docker0",
```

```bash
$ ifconfig | grep docker
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
```

**解释**

Docker 使用 Linux 桥接，在宿主机虚拟一个 Docker 容器网桥（docker0），Docker 启动一个容器时会根据 Docker 网桥的网段分配给容器一个 IP 地址，称为 Container-IP，同时 Docker 网桥是每个容器的默认网关。因为在同一宿主机内的容器都接入同一个网桥，这样容器之间就能够通过容器的 Container-IP 直接通信。

`docker run`的时候，没有指定 network 的话默认使用的网桥模式就是 bridge，使用的就是 docker0。在宿主机 ifconfig，就可以看到 docker0 和自己 create 的 network eth0，eth1，eth2……代表网卡一，网卡二，网卡三……，lo代表 127.0.0.1，即 localhost，inet addr 用来表示网卡的 IP 地址。

网桥 docker0 会创建一对对等虚拟设备接口一个叫 veth，另一个叫 eth0，成对匹配。

- 整个宿主机的网桥模式都是 docker0，类似一个交换机有一堆接口，每个接口叫 veth，在本地主机和容器内分别创建一个虚拟接口，并让他们彼此联通（这样一对接口叫 veth pair）
- 每个容器实例内部也有一块网卡，每个接口叫 eth0
- docker0 上面的每个 veth 匹配某个容器实例内部的 eth0，两两配对，一一匹配

 通过上述，将宿主机上的所有容器都连接到这个内部网络上，两个容器在同一个网络下,会从这个网关下各自拿到分配的 ip，此时两个容器的网络是互通的。

**验证**

先运行两个微服务

```bash
$ docker run -d -p 8081:8080 --name micro1 microservice:1.0
$ docker run -d -p 8082:8080 --name micro2 microservice:1.0
```

在主机上查看 IP 地址情况

```bash
$ ip addr | tail -n 8
33: vethc4d4e89@if32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether a2:8c:2e:08:09:61 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::a08c:2eff:fe08:961/64 scope link 
       valid_lft forever preferred_lft forever
35: veth4aafe45@if34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether 96:e2:34:87:e1:88 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::94e2:34ff:fe87:e188/64 scope link 
       valid_lft forever preferred_lft forever
```

分别在容器内部查看 IP 地址情况，可能需要安装`apt-get install -y iproute2`

```bash
$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
32: eth0@if33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

```bash
$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
34: eth0@if35: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

可以发现主机上的`33: vethc4d4e89@if32`匹配 micro1 的`32: eth0@if33`，`35: veth4aafe45@if34`匹配 micro2 的`34: eth0@if35`。

#### 2.hoat

直接使用宿主机的 IP 地址与外界进行通信，不再需要额外进行NAT 转换。容器将不会获得一个独立的 Network Namespace，而是和宿主机共用一个 Network Namespace。容器将不会虚拟出自己的网卡而是使用宿主机的 IP 和端口。

因为直接映射到宿主机的端口，所以`host`不能配合`-p`一起使用，例如下面的命令将会产生警告：

```bash
$ docker run -d -p 8083:8080 --network host --name micro microservice:1.0
WARNING: Published ports are discarded when using host network mode
d2dee142f94dc505d019c2912454cf8cc2996443050026ae7f902fb22206190c
```

此时`-p`设置的参数不会起到任何作用，端口号是微服务默认的端口号`8080`。

启动时直接使用下面的命令即可：

```bash
$ docker run -d --network host --name micro microservice:1.0
```

此时查看 micro 容器的网络情况：

```bash
$ docker inspect micro | tail -n 20
            "Networks": {
                "host": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "d57374a995d721d98ea3393b4c688a70a8747b31ddc2d68361a176276a549e94",
                    "EndpointID": "f99a63966c72e80baa45842d497c0dede0eac9741ab7522eeb4bd6c58ad77955",
                    "Gateway": "",
                    "IPAddress": "",
                    "IPPrefixLen": 0,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "",
                    "DriverOpts": null
                }
            }
        }
    }
]
```

可以发现没有使用网关和分配 IP 地址，因为使用的是宿主机的 IP 地址和端口，即外部主机与容器可以直接通信，从外部访问时直接访问`http://宿主机IP:8080`即可。

```bash
$ curl http://172.16.16.128:8080/hello
hello world
```

#### 3.none

在此模式下，并不为 Docker 容器进行任何网络配置，即这个 Docker 容器没有网卡、IP、路由等信息，只有一个 lo（表示本地回环），需要我们自己为 Docker 容器添加网卡、配置 IP 等。

```bash
$ docker run -d --network none --name micro microservice:1.0
4557daa3219dbfff2acae5d98f8c24abd711423557ba0dc48e31e0e47df30459
$ docker inspect micro | tail -n 20
            "Networks": {
                "none": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "c03ed094b7097d0cde29369c81790fb0df0309418a6ff55863465e1e32427a25",
                    "EndpointID": "cda18dd88a08e4e69305352d38f04826b939ea4b8d82a095c208f2c73f659951",
                    "Gateway": "",
                    "IPAddress": "",
                    "IPPrefixLen": 0,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "",
                    "DriverOpts": null
                }
            }
        }
    }
]
```

#### 4.container

新建的容器和已经存在的一个容器共享一个网络 IP 配置而不是和宿主机共享。新创建的容器不会创建自己的网卡，配置自己的IP，而是和一个指定的容器共享 IP、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202301250314194.png" alt="image-20230121190827447" style="zoom:67%;" />

此时要注意新建的容器所暴露的端口不能和原容器相同，否则会出错：

```bash
$ docker run -it -p 8080:8080 --name alpine1 alpine /bin/sh
8b10dd18d9e5eadc30ef64182d33a7efc0e824b7a8242f95587200728df3ca64
$ docker run -it -p 8081:8080 --network container:alpine1 --name alpine1 alpine /bin/sh
docker: Error response from daemon: conflicting options: port publishing and the container type network mode.
See 'docker run --help'.
```

> **Alpine**
>
> Alpine Linux 是一款独立的、非商业的通用 Linux 发行版，专为追求安全性、简单性和资源效率的用户而设计。 可能很多人没听说过这个 Linux 发行版本，但是经常用 Docker 的朋友可能都用过，因为他小，简单，安全而著称，所以作为基础镜像是非常好的一个选择，可谓是麻雀虽小但五脏俱全，镜像非常小巧，不到 6M的大小，所以特别适合容器打包。

**验证共用搭桥**

```bash
$ docker run -it --name alpine1 alpine /bin/sh
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
16: eth0@if17: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

```bash
$ docker run -it --network container:alpine1 --name alpine2 alpine /bin/sh
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
16: eth0@if17: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

可以发现，alpine1 和 alpine2 共用同一个`eth`。

如果此时关闭 alpine1，再观察 alpine2:

```bash
$ docker stop alpine1
alpine1
$ docker exec -it alpine2 ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
```

可以发现`eth0`消失了。

#### 5.自定义网络

> `link`模式已过时

如果我们正常创建两个容器使用`bridge`网络，并在容器中分别 ping 另一容器的 IP 地址是可以 ping 通的，但是如果按照容器名是 ping 不同的。这样当 IP 发生变动时，就会变得很麻烦。

**ping IP**

```bash
$ docker run -it --name alpine1 alpine /bin/sh
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
18: eth0@if19: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
/ # ping 172.17.0.3
PING 172.17.0.3 (172.17.0.3): 56 data bytes
```

```bash
$ docker run -it --name alpine2 alpine /bin/sh
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
20: eth0@if21: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
/ # ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.322 ms
```

**ping 容器名**

```bash
ping alpine2
ping: bad address 'alpine2'
```

```bash
/ # ping alpine1
ping: bad address 'alpine1'
```

使用自定义桥接网络可以通过容器名访问相应容器，自定义网络默认使用的是桥接网络 bridge。

**新建自定义网络**

```bash
$docker network create my_net
24e287841354b6f99a13ef4606956550fab5f740a8ff0f5d8e658712c7ad3afd
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
2a452f365325   bridge    bridge    local
d57374a995d7   host      host      local
24e287841354   my_net    bridge    local
c03ed094b709   none      null      local
```

**新建容器使用自定义网络**

```bash
$ docker run -it --network my_net --name alpine1 alpine /bin/sh
$ docker run -it --network my_net --name alpine2 alpine /bin/sh
```

**测试 ping 容器名**

```bash
/ # ping alpine2
PING alpine2 (172.18.0.3): 56 data bytes
64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.295 ms
```

```bash
/ # ping alpine1
PING alpine1 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.347 ms
```

可以发现，自定义网络维护了主机名和 ip 的对应关系（ip 和域名都能通）。

**指定静态 IP**

要想给容器指定静态 IP，需要在创建自定义网络时指定`subnet`

```bash
docker network create --subnet 177.18.20.0/24 mynet
```

为容器分配内部静态 IP

```bash
docker run -d --network mynet --ip 177.18.20.10 centos
```

## 4.Docker-Compose容器编排

Compose 是 Docker 公司推出的一个工具软件，可以管理多个 Docker 容器组成一个应用。你需要定义一个 YAML 格式的配置文件 docker-compose.yml，写好多个容器之间的调用关系。然后，只要一个命令，就能同时启动/关闭这些容器。

官方文档：https://docs.docker.com/compose/compose-file/compose-file-v3/

安装卸载：https://docs.docker.com/compose/install/

> **注意**
>
> 选择相应的架构！

### 4.1 核心概念

- `docker-compose.yml`
- 服务：一个个应用容器实例，比如订单微服务、库存微服务、mysql 容器、nginx 容器或者 redis 容器
- 工程：由一组关联的应用容器组成的一个完整业务单元，在 docker-compose.yml 文件中定义

### 4.2 使用步骤

- 编写 Dockerfile 定义各个微服务应用并构建出对应的镜像文件
- 使用 docker-compose.yml 定义一个完整业务单元，安排好各个容器服务
- 执行`docker-compose up`命令启动整个应用程序

### 4.3 常用命令

| 命令 | 作用 |
| ---- | ---- |
|`docker-compose -h`|              查看帮助|
|`docker-compose up`| 启动所有 docker-compose 服务 |
|`docker-compose up -d`| 启动所有 docker-compose 服务并后台运行 |
|`docker-compose down`|             停止并删除容器、网络、卷、镜像|
|`docker-compose exec yml里面的服务id`| 进入容器实例内部`docker-compose exec docker-compose.yml文件中写的服务id /bin/bash` |
|`docker-compose ps`| 展示当前 docker-compose 编排过的运行的所有容器 |
|`docker-compose top`| 展示当前 docker-compose 编排过的容器进程 |
|`docker-compose logs yml里面的服务id`|   查看容器输出日志|
|`docker-compose config`|   检查配置|
|`docker-compose config -q`| 检查配置，有问题才有输出|
|`docker-compose restart`|  重启服务|
|`docker-compose start`|   启动服务|
|`docker-compose stop`|    停止服务|

### 4.4 编排微服务

参考文档：https://yeasy.gitbook.io/docker_practice/compose/compose_file

编写 docker-compose.yml 文件

```yaml
version: "3"

services:
  microService-fromDockerFile:
    # 可通过 build 指定 Dockerfile 的路径
    build:
    	context: ./dir
    	dockerfile: Dockerfile
    	args:
    	  arg1: 1
  # 相当于：docker run -p 6001:6001 -v /app/microService:/data --network atguigu_net --name ms01 zzyy_docker1:6
  microService:
    image: zzyy_docker:1.6
    # 不指定名称则会使用“路径前缀 + 定义的 service 名 + 数字后缀“作为容器名称
    container_name: ms01
    ports:
      - "6001:6001"
    volumes:
      - /app/microService:/data
    networks: 
      - atguigu_net 
    depends_on: 
      - redis
      - mysql
 
  redis:
    image: redis:6.0.8
    ports:
      - "6379:6379"
    volumes:
      - /app/redis/redis.conf:/etc/redis/redis.conf
      - /app/redis/data:/data
    networks: 
      - atguigu_net
    command: redis-server /etc/redis/redis.conf
 
  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: '123456'
			MYSQL_ALLOW_EMPTY_PASSWORD: 'no'
      MYSQL_DATABASE: 'db2021'
      MYSQL_USER: 'zzyy'
      MYSQL_PASSWORD: 'zzyy123'
    ports:
       - "3306:3306"
    volumes:
       - /app/mysql/db:/var/lib/mysql
       - /app/mysql/conf/my.cnf:/etc/my.cnf
       - /app/mysql/init:/docker-entrypoint-initdb.d
    networks:
      - atguigu_net
    command: --default-authentication-plugin=mysql_native_password #解决外部无法访问

# 相当于：docker network create atguigu_net
networks: 
   atguigu_net: 
```

由于都部署在同一个 docker 中，因此微服务访问数据库时可以使用 network 而不用 ip：

```properties
spring.datasource.url=jdbc:mysql://mysql:3306/db2021?
...
spring.redis.host=redis
```

执行`docker compose up`或者`docker compose up -d`即可同时启动三个容器，执行`docker-compose stop`即可同时停止三个容器。

## 5.可视化工具Portainer

Portainer 是一款轻量级的应用，它提供了图形化界面，用于方便地管理 Docker 环境，包括单机环境和集群环境。

官网：https://www.portainer.io/

通过命令行安装

```bash
$ docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
```

第一次登录需创建 admin 密码，访问地址：https://172.16.16.128:9443

## 6.容器监控

CAdvisor + influxDB + Granfana
