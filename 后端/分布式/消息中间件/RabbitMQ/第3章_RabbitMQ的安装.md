# 第3章_RabbitMQ的安装

## 1.RabbitMQ入门及安装

### 1.1 概述

与 Spring 同一家公司，与 Spring 的整合比较完整。

官网：https://www.rabbitmq.com/

什么是 RabbitMQ，官方给出来这样的解释：

> RabbitMQ is the most widely deployed open source message broker.
>
> With tens of thousands of users, RabbitMQ is one of the most popular open source message brokers. From T-Mobile to Runtastic, RabbitMQ is used worldwide at small startups and large enterprises.
>
> RabbitMQ is lightweight and easy to deploy on premises and in the cloud. It supports multiple messaging protocols. RabbitMQ can be deployed in distributed and federated configurations to meet high-scale, high-availability requirements.
>
> RabbitMQ runs on many operating systems and cloud environments, and provides a wide range of developer tools for most popular languages.
>
> 翻译以后：
>
> RabbitMQ 是部署最广泛的开源消息代理。
>
> RabbitMQ 拥有成千上万的用户，是最受欢迎的开源消息代理之一。从 T-Mobile 到 Runtastic，RabbitMQ 在全球范围内的小型初创企业和大型企业中都得到使用。
>
> RabbitMQ 轻巧，易于在内部和云中部署。它支持多种消息传递协议。RabbitMQ 可以部署在分布式和联合配置中，以满足大规模，高可用性的要求。
>
> RabbitMQ 可在许多操作系统和云环境上运行，并为大多数流行语言提供了广泛的开发人员工具。

简单概述：

RabbitMQ 是一个开源的遵循 AMQP 协议实现的基于 Erlang 语言编写，支持多种客户端（语言）。用于在分布式系统中存储消息，转发消息，具有高可用，高可扩性，易用性等特征。

### 1.2 安装RabbitMQ

1：下载地址：https://www.rabbitmq.com/download.html

2：环境准备：CentOS7.x + / Erlang

RabbitMQ 是采用 Erlang 语言开发的，所以系统环境必须提供 Erlang 环境，第一步就是安装 Erlang。

> erlang 和 RabbitMQ 版本的对照比较：https://www.rabbitmq.com/which-erlang.html

### 1.3 Erlang安装

查看系统版本号：

```bash
[root@iZm5eauu5f1ulwtdgwqnsbZ ~]# lsb_release -a
LSB Version:    :core-4.1-amd64:core-4.1-noarch
Distributor ID: CentOS
Description:    CentOS Linux release 8.3.2011
Release:        8.3.2011
Codename:       n/a
```

1. 安装下载

   参考地址：https://www.erlang-solutions.com/downloads/

   下载后放到`/usr/local/rabbitmq`中，再执行下面命令解压：

   ```bash
   wget https://packages.erlang-solutions.com/erlang-solutions-2.0-1.noarch.rpm
   rpm -Uvh erlang-solutions-2.0-1.noarch.rpm
   ```

2. 安装

   ```bash
   yum install -y erlang
   ```

3. 查看版本号

   ```bash
   erl -v
   ```

4. 安装socat

   ```bash
   yum install -y socat
   ```

5. 安装rabbitmq

   下载地址：https://www.rabbitmq.com/download.html

   - 下载后放到`/usr/local/rabbitmq`中，再执行下面命令解压：

     ```bash
     rpm -Uvh rabbitmq-server-3.10.1-1.el8.noarch.rpm
     ```

   - 启动 rabbitmq 服务

     ```bash
     # 启动服务
     > systemctl start rabbitmq-server
     # 查看服务状态
     > systemctl status rabbitmq-server
     # 停止服务
     > systemctl stop rabbitmq-server
     # 开机启动服务
     > systemctl enable rabbitmq-server
     ```

6. RabbitMQ的配置

   RabbitMQ 默认情况下有一个配置文件，定义了 RabbitMQ 的相关配置信息，默认情况下能够满足日常的开发需求。如果需要修改需要，需要自己创建一个配置文件进行覆盖。

   参考官网：

   - https://www.rabbitmq.com/documentation.html
   - https://www.rabbitmq.com/configure.html
   - https://www.rabbitmq.com/configure.html#config-items
   - https://github.com/rabbitmq/rabbitmq-server/blob/add-debug-messages-to-quorum_queue_SUITE/docs/rabbitmq.conf.example

   相关端口

   > 5672：RabbitMQ 的通讯端口
   >
   > 25672：RabbitMQ 的节点间的 CLI 通讯端口
   >
   > 15672：RabbitMQ HTTP_API 的端口，管理员用户才能访问，用于管理 RabbitMQ，需要启动 Management 插件
   >
   > 1883、8883：MQTT 插件启动时的端口
   >
   > 61613、61614：STOMP 客户端插件启用的时候的端口
   >
   > 15674、15675：基于 webscoket 的 STOMP 端口和 MOTT 端口

一定要注意：RabbitMQ 在安装完毕以后，会绑定一些端口，如果你购买的是阿里云或者腾讯云相关的服务器一定要在安全组中把对应的端口添加到防火墙。

## 2.RabbitMQWeb管理界面及授权操作

### 2.1 RabbitMQ管理界面

默认情况下，rabbitmq 是没有安装 web 端的客户端插件，需要安装才可以生效

```bash
rabbitmq-plugins enable rabbitmq_management
```

> **说明**
>
> rabbitmq 有一个默认账号和密码是`guest`，默认情况只能在 localhost 本机下访问，所以需要添加一个远程登录的用户。
>
> <img src="img/image-20220513222854311.png" alt="image-20220513222854311" style="zoom:67%;" />

安装完毕以后，重启服务即可。

```bash
systemctl restart rabbitmq-server
```

一定要记住，在对应服务器（阿里云，腾讯云等）的安全组中开放 15672 的端口。

在浏览器访问 http://ip:15672/ 如下：

<img src="img/image-20220513222959883.png" alt="image-20220513222959883" style="zoom:67%;" />

### 2.2 授权账号和密码

新增用户

```bash
rabbitmqctl add_user admin admin
```

设置用户分配操作权限

```bash
rabbitmqctl set_user_tags admin administrator
```

用户级别：

- `administrator`：可以登录控制台、查看所有信息、可以对 rabbitmq 进行管理
- `monitoring`：监控者登录控制台，查看所有信息
- `policymaker`：策略制定者登录控制台，指定策略
- `managment`：普通管理员，登录控制台

如果需要为其他用户添加资源权限，可以执行下面的命令：

```bash
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```

### 2.3 小结

```bash
# 增加账户
rabbitmqctl add_user 账号 密码
# 设置管理员权限
rabbitmqctl set_user_tags 账号 administrator
# 修改密码
rabbitmqctl change_password Username Newpassword
# 删除用户
rabbitmqctl delete_user Username
# 查看用户清单
rabbitmqctl list_users
# 为用户设置 administrator 角色
rabbitmqctl set_permissions -p / 用户名 ".*" ".*" ".*"
rabbitmqctl set_permissions -p / root ".*" ".*" ".*"
```

## 3.RabbitMQ之Docker安装

### 3.1 Docker安装RabbitMQ

**虚拟化容器技术—Docker的安装**

（1）yum 包更新到最新

```bash
yum update
```

（2）安装需要的软件包， yum-util 提供 yum-config-manager 功能，另外两个是 devicemapper 驱动依赖的

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
```

（3）设置 yum 源为阿里云

```Bash
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

（4）安装 docker

```bash
yum install docker-ce -y
```

（5）安装后查看 docker 版本

```bash
docker -v
```

（6）安装加速镜像

```bash
 sudo mkdir -p /etc/docker
 sudo tee /etc/docker/daemon.json <<-'EOF'
 {
"registry-mirrors": ["https://0wrdwnn6.mirror.aliyuncs.com"]
 }
 EOF
 sudo systemctl daemon-reload
 sudo systemctl restart docker
```

**docker的相关命令**

```bash
# 启动docker：
systemctl start docker

# 停止docker：
systemctl stop docker

# 重启docker：
systemctl restart docker

# 查看docker状态：
systemctl status docker

# 开机启动：  
systemctl enable docker
systemctl unenable docker

# 查看docker概要信息
docker info

# 查看docker帮助文档
docker --help
```

**安装rabbitmq**

参考网站：

- https://www.rabbitmq.com/download.html
- https://registry.hub.docker.com/_/rabbitmq/

**获取rabbit镜像**

```bash
docker pull rabbitmq:management
```

**创建并运行容器**

```bash
docker run -di --name myrabbit -p 15672:15672 rabbitmq:management
```

- hostname：指定容器主机名称
- name：指定容器名称
- -p：将 mq 端口号映射到本地

或者运行时设置用户和密码，此时也会自动获取 rabbit 镜像

```bash
docker run -di --name myrabbit -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin -p 15672:15672 -p 5672:5672 -p 25672:25672 -p 61613:61613 -p 1883:1883 rabbitmq:management
```

查看日志

```bash
docker logs -f myrabbit
```

**容器运行正常**

使用`http://你的IP地址:15672`访问 rabbit 控制台

### 3.2 额外Linux相关排查命令

```bash
# 查看日记信息
more xxx.log
# 查看端口是否被占用
netstat -naop | grep 5672
# 查看进程
ps -ef | grep 5672
# 服务
systemctl stop
```

## 