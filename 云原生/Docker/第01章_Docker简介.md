# 第01章_Docker入门与安装

## 1.简介

Docker 是一种容器虚拟化技术，方便做持续集成并有助于整体发布。

Docker 是基于 Go 语言实现的云开源项目。Docker 的主要目标是`Build，Ship and Run Any App,Anywhere`，即一次镜像，处处运行，通过对应用组件的封装、分发、部署、运行等生命周期的管理，实现了跨平台、跨服务器。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230109153105118-9c688ff9385960cd114408b621ca3297-052f0f.png" alt="image-20230109153105118" style="zoom: 67%;" />

### 1.1 Docker的优点

- 一次构建、随处运行

- 更快速的应用交付和部署

- 更便捷的升级和扩缩容

- 更简单的系统运维

- 更高效的计算资源利用

  Docker 是内核级虚拟化，仅包含业务运行所需的运行环境，不像传统的虚拟化技术一样需要额外的 Hypervisor 支持，所以在一台物理机上可以运行很多个容器实例，可大大提升物理服务器的 CPU 和内存的利用率。

### 1.2 Docker的基本组成

Docker 本身是一个容器运行载体或称之为管理引擎。我们把应用程序和配置依赖打包好形成一个可交付的运行环境，这个打包好的运行环境就是 image 镜像文件。只有通过这个镜像文件才能生成 Docker 容器实例。image 文件可以看作是容器的模板。Docker 根据 image 文件生成容器的实例。同一个 image 文件，可以生成多个同时运行的容器实例。

**镜像**

Docker 镜像（Image）就是一个**只读**的模板。镜像可以用来创建 Docker 容器，一个镜像可以创建很多容器。它也相当于是一个 root 文件系统。比如官方镜像 centos:7 就包含了完整的一套 centos:7 最小系统的 root 文件系统。它包含运行某个软件所需的所有内容，我们把应用程序和配置依赖打包好形成一个可交付的运行环境（包括代码、运行时需要的库、环境变量和配置文件等），这个打包好的运行环境就是镜像文件。

> **Docker 镜像加载原理**
>
> docker 的镜像实际上由一层一层的文件系统组成，这种层级的文件系统 UnionFS。镜像分层最大的一个好处就是共享资源，方便复制迁移，就是为了复用。
>
> 比如说有多个镜像都从相同的 base 镜像构建而来，那么 Docker Host 只需在磁盘上保存一份 base 镜像，同时内存中也只需加载一份 base 镜像，就可以为所有容器服务了。而且镜像的每一层都可以被共享。
>
> **bootfs（boot file system）**
>
> 主要包含`bootloader`和`kernel`，`bootloader`主要是引导加载`kernel`，Linux 刚启动时会加载 bootfs 文件系统，在 Docker 镜像的最底层是引导文件系统 bootfs。这一层与我们典型的 Linux/Unix 系统是一样的，包含 boot 加载器和内核。当 boot 加载完成之后整个内核就都在内存中了，此时内存的使用权已由 bootfs 转交给内核，此时系统也会卸载 bootfs。
>
> **rootfs（root file system）**
>
> 在 bootfs 之上。包含的就是典型 Linux 系统中的 /dev、/proc、/bin、/etc 等标准目录和文件。rootfs 就是各种不同的操作系统发行版，比如 Ubuntu，Centos 等等。 
>
> 对于一个精简的 OS，rootfs 可以很小，只需要包括最基本的命令、工具和程序库就可以了，因为底层直接用 Host 的 kernel，自己只需要提供 rootfs 就行了。对于不同的 linux 发行版，bootfs 基本是一致的，rootfs 会有差别，因此不同的发行版可以公用 bootfs。

> **UnionFS（联合文件系统）**
>
> Union 文件系统（UnionFS）是一种分层、轻量级并且高性能的文件系统，它支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下（unite several directories into a single virtual filesystem）。Union 文件系统是 Docker 镜像的基础。镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。
>
> 特性：一次同时加载多个文件系统，但从外面看起来，只能看到一个文件系统，联合加载会把各层文件系统叠加起来，这样最终的文件系统会包含所有底层的文件和目录。

Docker 镜像层都是只读的，容器层是可写的。当容器启动时，一个新的可写层被加载到镜像的顶部。 这一层通常被称作“容器层”，“容器层”之下的都叫“镜像层”。当容器启动时，一个新的可写层被加载到镜像的顶部。这一层通常被称作“容器层”，“容器层”之下的都叫“镜像层”。所有对容器的改动 - 无论添加、删除、还是修改文件都只会发生在容器层中。只有容器层是可写的，容器层下面的所有镜像层都是只读的。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230115233542948-3ae7bff9ddab20891cf0509f64539659-0a1e3c.png" alt="image-20230115233542948" style="zoom: 50%;" />

Docker 中的镜像分层，支持通过扩展现有镜像，创建新的镜像。类似 Java 继承于一个 Base 基础类，自己再按需扩展。新镜像是从 base 镜像一层一层叠加生成的。每安装一个软件，就在现有镜像的基础上增加一层。

![image-20230115233901788](https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230115233901788-f2e9ca648c0e7a220043af5d8c099bee-51047e.png)

**容器**

可以把容器看做是一个简易版的 Linux 环境（包括 root 用户权限、进程空间、用户空间和网络空间等）和运行在其中的应用程序。

**仓库**

仓库（Repository）是集中存放镜像文件的场所。仓库分为公开仓库（Public）和私有仓库（Private）两种形式。最大的公开仓库是 Docker Hub，存放了数量庞大的镜像供用户下载。国内的公开仓库包括阿里云 、网易云等。

### 1.3 Docker平台架构

Docker 是一个 C/S 模式的架构，后端是一个松耦合架构，运行的基本流程为：

1. 用户通过 Client 与 Docker Daemon 建立通信并发送请求
2. Docker Daemon 提供 Docker Server 使其可以接受 Client 的请求
3. Docker Engine 执行 Docker 内部的一系列工作，每一项工作都是以一个 Job 形式存在
4. Job 运行过程中，当需要容器镜像时，从 Docker Registry 中下载镜像，并通过镜像管理驱动 Graph Driver 将镜像以 Graph 的形式存储
5. 当需要为 Docker 创建网络环境时，通过网络管理驱动 Network Driver 创建并配置 Docker 容器网络环境
6. 通过 Exec Driver 限制 Docker 容器运行资源或执行用户指令等操作
7. Libcontainer 是一项独立的容器管理包，Network Driver 以及 Exec Driver 都是通过 Libcontainer 来实现具体对容器进行的操作

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230109162918752-cae0b6d9d5d60e98c56b1cbb0b6460d0-2df30e.png" alt="image-20230109162918752" style="zoom:80%;" />

## 2.Docker的下载与安装

官网：http://www.docker.com

安装镜像的仓库：https://hub.docker.com/

### 2.1 Docker的安装与卸载

Docker 并非是一个通用的容器工具，它依赖于 Linux 内核环境。实质上是在 Linux 下制造了一个隔离的文件环境，执行效率等同于 Linux 主机。

> **Linux 下查看内核信息**
>
> `cat /etc/redhat-release`查看 Linux 信息，`uname`查看当前系统相关信息。

**安装步骤**

https://docs.docker.com/engine/install/centos/

- 卸载旧版本

  ```bash
  sudo yum remove docker \
                    docker-client \
                    docker-client-latest \
                    docker-common \
                    docker-latest \
                    docker-latest-logrotate \
                    docker-logrotate \
                    docker-engine
  ```

- `yum`安装 gcc 相关

  ```bash
  yum -y install gcc
  yum -y install gcc-c++
  ```

- 安装 repository

  ```bash
  sudo yum install -y yum-utils
  sudo yum-config-manager \
      --add-repo \
      https://download.docker.com/linux/centos/docker-ce.repo
  ```

  > 国内使用阿里云镜像
  >
  > ```bash
  > sudo yum-config-manager \
  > 	--add-repo \
  > 	http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
  > ```

- 更新 yum 软件包索引（可选）

  ```bash
  yum makecache fast
  ```

- 安装 Docker CE

  ```bash
  sudo yum -y install docker-ce docker-ce-cli containerd.io docker-compose-plugin
  ```

- 开启 Docker

  ```bash
  sudo systemctl start docker
  ```

- 测试

  ```bash
  docker version
  docker run hello-world
  ```

**卸载步骤**

```bash
systemctl stop docker
sudo yum remove docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

### 2.2 阿里云镜像加速

https://promotion.aliyun.com/ntms/act/kubernetes.html

- 注册一个属于自己的阿里云账户

- 获得加速器地址连接

  - 登陆阿里云开发者平台，点击控制台
  - 选择容器镜像服务
  - 打开镜像工具 - 镜像加速器
  - 复制脚本，在 bash 执行

- 重启服务器

  ```bash
  ystemctl daemon-reload
  systemctl restart docker
  ```

## 3.run命令底层流程

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230109214708010-ec739d0af033be4f8a2841619d0f051e-d9b62d.png" alt="image-20230109214708010"  />

## 4.容器和虚拟机

**区别**

- 传统虚拟机技术是虚拟出一套硬件后，在其上运行一个完整操作系统，在该系统上再运行所需应用进程

- 容器内的应用进程直接运行于宿主的内核，容器内没有自己的内核且也没有进行硬件虚拟。因此容器要比传统虚拟机更为轻便

- 每个容器之间互相隔离，每个容器有自己的文件系统 ，容器之间进程不会相互影响，能区分计算资源

**Docker 会比 VM 虚拟机快**

1. docker 有着比虚拟机更少的抽象层

   由于 docker 不需要 Hypervisor（虚拟机）实现硬件资源虚拟化，运行在 docker 容器上的程序直接使用的都是实际物理机的硬件资源。因此在 CPU、内存利用率上 docker 将会在效率上有明显优势。

2. docker 利用的是宿主机的内核，而不需要加载操作系统 OS 内核

   当新建一个容器时，docker 不需要和虚拟机一样重新加载一个操作系统内核。当新建一个虚拟机时，虚拟机软件需要加载 OS，返回新建过程是分钟级别的。而 docker 由于直接利用宿主机的操作系统，则省略了返回过程，因此新建一个 docker 容器只需要几秒钟。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202301281534918.png" alt="image-20200505183738289" style="zoom:80%;" />