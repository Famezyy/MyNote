# 第02章_Docker命令

## 1.帮助启动类命令

```bash
# 启动 docker
systemctl start docker
# 停止 docker
systemctl stop docker
# 重启 docker
systemctl restart docker
# 查看 docker 状态
systemctl status docker
# 开机启动
systemctl enable docker
# 查看 docker 概要信息
docker info
# 查看 docker 总体帮助文档
docker --help
# 查看 docker 命令帮助文档
docker 具体命令 --help
```

## 2.镜像命令

### 2.1 列出本地上的镜像

`docker images [option]`

```bash
[root@myServer1 ~]# docker images
REPOSITORY    TAG       IMAGE ID       CREATED         SIZE
hello-world   latest    feb5d9fea6a5   15 months ago   13.3kB
```

- REPOSITORY：表示镜像的仓库源

- TAG：镜像的标签版本号

  同一仓库源可以有多个 TAG 版本，代表这个仓库源的不同个版本，我们使用 REPOSITORY:TAG 来定义不同的镜像。如果你不指定一个镜像的版本标签，例如你只使用 ubuntu，docker 将默认使用 ubuntu:latest 镜像

- IMAGE ID：镜像 ID

- CREATED：镜像创建时间

- SIZE：镜像大小

**OPTIONS 说明**

- `-a`：列出本地所有的镜像（含历史映像层）

- `-q`：只显示镜像 ID

  `-f`：过滤

### 2.2 查找镜像

`docker search [option] [image name]`

```bash
[root@myServer1 ~]# docker search redis
NAME                                DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
redis                               Redis is an open source key-value store that…   11707     [OK]       
bitnami/redis                       Bitnami Redis Docker Image                      236                  [OK]
redislabs/redisinsight              RedisInsight - The GUI for Redis                77                   
redislabs/redisearch                Redis With the RedisSearch module pre-loaded…   56                   
...
```

- NAME：镜像名称
- DESCRIPTION：镜像说明
- STARS：点赞说明
- OFFICIAL：是否为官方
- AUTOMATED：是否是自动创建

**OPTIONS 说明**

- `--limit`：只列出 N 个镜像，默认 25 个

### 2.3 下载镜像

`docker pull [image name]`，可以带上 TAG 或者不带：

- `docker pull imageName:TAG`

- `docker pull imageName`，等同于`docker pull imageName:latest`

### 2.4 查看镜像、容器、数据卷所占用的空间

`docker system df`

### 2.5 删除镜像

`docker rmi [image name/image id]`

- 删除单个：`docker rmi imageId`

  ```bash
  [root@myServer1 ~]# docker rmi -f hello-world
  Untagged: hello-world:latest
  Untagged: hello-world@sha256:94ebc7edf3401f299cd3376a1669bc0a49aef92d6d2669005f9bc5ef028dc333
  Deleted: sha256:feb5d9fea6a5e9606aa995e879d862b825965ba48de054caab5ef356dc6b3412
  ```

- 删除多个：`docker rmi -f imageName:TAG imageName:TAG ...`

- 删除全部：`docker rmi -f ${docker images -qa}`

### 2.6 提交镜像

`docker commit -m="message" -a="author" [container id] [image name][:tag]`

提交容器副本使之成为一个新的镜像。所有改动都会被保存，同`export`和`import`命令。

### 2.7 发布到阿里云

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230119213036399-7c77f043af85167eb6a71d74f1182b57-eb1c80.png" alt="image-20230119213036399" style="zoom:80%;" />

阿里云开发者平台：https://promotion.aliyun.com/ntms/act/kubernetes.html

官方Docker Hub地址：https://hub.docker.com/

**创建镜像仓库**

容器镜像服务 - 个人版 - 创建命名空间 - 创建镜像仓库（最后一步选择本地仓库）- 访问凭证设置密码

**推送与拉取**

进入镜像仓库，复制相应的推送脚本执行

### 2.8 发布到私有库

Docker Registry 是官方提供的工具，可以用于构建私有镜像仓库。

- 下载镜像 Docker Registry

  ```bash
  $ docker pull registry
  ```

- 运行私有库 Registry，相当于本地有个私有 Docker hub

  ```bash
  $ $ docker run -d \
      -p 5000:5000 \
      -v /opt/data/registry:/var/lib/registry \
      registry
  ```

  > **注意**
  >
  > 默认情况，仓库被创建在容器的`/var/lib/registry`目录下，建议自行用容器卷映射，方便于宿主机联调。

- `curl`命令查看私服库上有什么镜像

  ```bash
  $ curl -XGET http://127.0.0.1:5000/v2/_catalog
  {"reporsitories":[]}
  # 查看 image 的版本号 list
  $ curl -XGET http://127.0.0.1:5000/v2/[image]/tags/list
  ```

- 将镜像 myUbuntu:1.2 修改成符合私服规范的 Tag

  ```bash
  $ docker tag [image:Tag] Registry_Host:Registry_Port/Repository:Tag
  ```

  例：

  ```bash
  $ docker tag  myUbuntu:1.2  192.168.11.101:5000/myUbuntu:1.0
  ```

- docker 默认不允许 http 方式推送镜像，通过配置选项来取消这个限制：修改配置文件使之支持 http

  ```bash
  $ vim /etc/docker/daemon.json
  # 添加如下配置
  {
    "registry-mirrors": ["https://aa25jngu.mirror.aliyuncs.com"],
    "insecure-registries": ["192.168.11.101:5000"]
  }
  ```

  > **注意**
  >
  > registry-mirrors 配置的是国内阿里提供的镜像加速地址，不用加速的话访问官网的会很慢。
  >
  > 另外建议修改配置后重启 docker：`systemctl restart docker`

- `push`推送到私服库

  ```bash
  $ docker push 127.0.0.1:5000/myUbuntu:1.0
  ```

- `pull`到本地并运行

  ```bash
  $ docker pull 192.168.111.162:5000/myUbuntu:1.0
  ```

> **配置高级仓库**
>
> https://yeasy.gitbook.io/docker_practice/repository/nexus3_registry

### 面试题：docker 虚悬镜像

仓库名、标签都是\<none\>的镜像，俗称虚悬镜像 dangling image。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230109225918706-fe135b483329b2e8cd7c21bdca45c136-9e2697.png" alt="image-20230109225918706" style="zoom:80%;" />

## 3.容器命令

有镜像才能创建容器，这是根本前提。因此首先`pull`一个 ubuntu 或者 centos 的镜像（只加载了内核相关组件）。

### 3.1 新建并启动容器

`docker run [OPTIONS] IMAGE [COMMAND] [ARG...]`

**OPTIONS 说明**

- `--name="newDockerName"`：为容器指定一个名称，不指定则随机分配

- `-d`：后台运行容器并返回容器 ID

  > 不是所有镜像都可以用`-d`来后台启动， 例如`docker run -d ubuntu`虽然可以执行成功，但容器会立即自行销毁，因为 Docker 容器后台运行时必须有一个前台进程。而`docker run -d redis`就可以启动成功。

- `-i`：以交互模式运行容器，通常与`-t`同时使用

- `-t`：为容器重新分配一个伪输入终端，通常与`-i`同时使用，也即启动交互式容器（前有伪终端，等待交互）

  > **启动交互式容器（前台命令行）**
  >
  > ```bash
  > $ docker run -it ubuntu /bin/bash
  > ```
  >
  > `/bin/bash`：放在镜像后的是命令，这条命令会打开交互式 Shell。同`bash`

- `-P`：随机端口映射，主机的映射端口会被随机分配

- `-p`：指定端口映射

  |              参数               |                说明                 |
  | :-----------------------------: | :---------------------------------: |
  |   `-p hostPort:containerPort`   |        端口映射`-p 8080:80`         |
  | `-p ip:hostPort:containerPort`  | 配置监听地址`-p 10.0.0.100:8080:80` |
  |     `-p ip::containerPort`      |   随机分配端口`-p 10.0.0.100::80`   |
  | `-p hostPort:containerPort:udp` |      指定协议`-p 8080:80:tcp`       |
  |      `-p 81:80 -p 443:443`      |              指定多个               |

- `--restart=always`：docker 重启后会自动重启该容器

### 3.2 列出当前正在运行的容器

`docker ps [OPTIONS]`

**OPTIONS 说明**

- `-a`：列出当前所有正在运行的容器和运行过的容器
- `-l`：显示最近创建的 1 个容器
- `-n [number]`：显示最近 number 个创建的容器
- `-q`：静默模式，只显示容器编号

### 3.3 退出容器

- `exit`：容器停止
- `ctrl+p+q`：容器不停止

### 3.4 启动已停止的容器

`docker start [container id or name]`

### 3.5 重启容器

`docker restart [container id or name]`

### 3.6 停止容器

`docker stop [container id or name]`

强制停止：`docker kill [container id or name]`

### 3.7 删除容器

`docker rm [container id or name]`

强制删除：`docker rm -f [container id or name]`

删除全部容器：`docker rm -f $(docker ps -aq)`or`docker ps -aq | xargs docker rm`

### 3.8 查看容器运行状况

- 查看日志：`docker logs [container id or name]`

- 查看容器内运行的进程：`docker top [container id or name]`

- 查看容器内部配置：`docker inspect [container id or name]`

### 3.9 交互正在运行的容器

进入正在运行的容器并以命令行交互：`docker exec -it [container id or name] bash`

后台运行命令：`docker exec -d [container id or name] [command]`

> **注意**
>
> 也可以用`docker attach [container id or name]`重新进入容器，但是此时会进入容器启动时绑定的的终端，用`exit`退出会结束该终端从而导致容器停止。而`exec`会开启新的终端，用`exit`退出不会导致容器停止。推荐每次使用`exec`。

**例**

首先后台启动一个 redis，`docker run -d redis`，或者`docker run -it redis`进去后再`ctrl+p+q`。

启动 redis 客户端：`docker exec -it [container id or name] redis-cli`

### 3.10 从容器拷贝文件

`docker cp [容器ID或名字:容器内的路径 目的主机路径]`

例：`docker cp esafr42:/usr/local/a.txt /tmp`

> **注意**
>
> 把文件路径反过来即可从主机拷贝文件到容器。

### 3.11 导入和导出容器

导出容器的内容为一个 tar 归档文件：`docker export [container id or name] > fileName.tar`

从 tar 包中创建一个新的镜像：`cat fileName.tar | docker import - [prefix/]imageName[:tag]`

### 3.12 总结

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230115222224465-4eafac09eb0af2b1736b092a5fc85784-3a5e8f.png" alt="image-20230115222224465"  />

```bash
attach    Attach to a running container                 # 当前 shell 下 attach 连接指定运行镜像
build     Build an image from a Dockerfile              # 通过 Dockerfile 定制镜像
commit    Create a new image from a container changes   # 提交当前容器为新的镜像
cp        Copy files/folders from the containers filesystem to the host path   #从容器中拷贝指定文件或者目录到宿主机中
create    Create a new container                        # 创建一个新的容器，同 run，但不启动容器
diff      Inspect changes on a container's filesystem   # 查看 docker 容器变化
events    Get real time events from the server          # 从 docker 服务获取容器实时事件
exec      Run a command in an existing container        # 在已存在的容器上运行命令
export    Stream the contents of a container as a tar archive   # 导出容器的内容流作为一个 tar 归档文件[对应 import ]
history   Show the history of an image                  # 展示一个镜像形成历史
images    List images                                   # 列出系统当前镜像
import    Create a new filesystem image from the contents of a tarball # 从tar包中的内容创建一个新的文件系统映像[对应export]
info      Display system-wide information               # 显示系统相关信息
inspect   Return low-level information on a container   # 查看容器详细信息
kill      Kill a running container                      # kill 指定 docker 容器
load      Load an image from a tar archive              # 从一个 tar 包中加载一个镜像[对应 save]
login     Register or Login to the docker registry server    # 注册或者登陆一个 docker 源服务器
logout    Log out from a Docker registry server          # 从当前 Docker registry 退出
logs      Fetch the logs of a container                 # 输出当前容器日志信息
port      Lookup the public-facing port which is NAT-ed to PRIVATE_PORT    # 查看映射端口对应的容器内部源端口
pause     Pause all processes within a container        # 暂停容器
ps        List containers                               # 列出容器列表
pull      Pull an image or a repository from the docker registry server   # 从docker镜像源服务器拉取指定镜像或者库镜像
push      Push an image or a repository to the docker registry server    # 推送指定镜像或者库镜像至docker源服务器
restart   Restart a running container                   # 重启运行的容器
rm        Remove one or more containers                 # 移除一个或者多个容器
rmi       Remove one or more images       # 移除一个或多个镜像[无容器使用该镜像才可删除，否则需删除相关容器才可继续或 -f 强制删除]
run       Run a command in a new container              # 创建一个新的容器并运行一个命令
save      Save an image to a tar archive                # 保存一个镜像为一个 tar 包[对应 load]
search    Search for an image on the Docker Hub         # 在 docker hub 中搜索镜像
start     Start a stopped containers                    # 启动容器
stop      Stop a running containers                     # 停止容器
tag       Tag an image into a repository                # 给源中镜像打标签
top       Lookup the running processes of a container   # 查看容器中运行的进程信息
unpause   Unpause a paused container                    # 取消暂停容器
version   Show the docker version information           # 查看 docker 版本号
wait      Block until a container stops, then print its exit code   # 截取容器停止时的退出状态值
```

## 4.容器数据卷

Docker 挂载主机目录访问如果出现`cannot open directory .: Permission denied`，解决办法：在挂载目录后多加一个`--privileged=true`参数即可，表示给容器 root 权限。

如果是 CentOS7 安全模块会比之前系统版本加强，不安全的会先禁止，所以目录挂载的情况被默认为不安全的行为，在 SELinux 里面挂载目录被禁止掉了，如果要开启，我们一般使用`--privileged=true`命令，扩大容器的权限解决挂载目录没有权限的问题，也即使用该参数，container 内的 root 拥有真正的 root 权限，否则，container 内的 root 只是外部的一个普通用户权限。

### 4.1 卷

指目录或文件，存在于一个或多个容器中，由 Docker 挂载到容器，但不属于联合文件系统，因此能够绕过 Union File System 提供一些用于持续存储或共享数据的特性。

Docker 容器产生的数据如果不备份，那么当容器实例删除后容器内的数据自然也就没有了。卷的设计目的就是数据的持久化，完全独立于容器的生存周期，因此 Docker 不会在容器删除时删除其挂载的数据卷。有点类似 Redis 里面的 rdb 和 aof 文件，将 docker 容器内的数据保存进宿主机的磁盘中。

特点：

- 数据卷可在容器之间共享或重用数据
- 卷中的更改可以直接实时生效（挂载的特性）
- 数据卷中的更改不会包含在镜像的更新中
- 数据卷的生命周期一直持续到没有容器使用它为止

运行一个带有容器卷存储功能的容器实例：

```bash
$ docker run -it --privileged=true -v [/宿主机绝对路径目录:/容器内目录][:ws] [镜像名]
```

### 4.2 案例

开启一个 ubuntu 容器，将容器内的`/tmp/myDockerData`目录挂载到主机的`/tmp/myHostData`目录：

```bash
$ docker run -it --privileged=true -v /tmp/myHostData:/tmp/myDockerData ubuntu bash
```

查看数据卷是否挂载成功

```bash
$ docker inspect [容器ID]
```

容器和宿主机之间数据共享：

- docker 修改，主机同步获得 

- 主机修改，docker 同步获得

- docker 容器 stop，主机修改，docker 容器重启看数据是否同步

### 4.3 读写规则

默认情况下启动时是读写都可（`ws`），需要限制容器实例内部只能读取不能写时：

```bash
$ docker run -it --privileged=true -v /宿主机绝对路径目录:/容器内目录:ro 镜像名
```

此时如果宿主机写入内容，可以同步给容器内，容器可以读取到。

### 4.4 继承和共享

映射的继承是相互独立的，不会因为 u1 的容器下线而失效。

- 容器 1 完成和宿主机的映射

  ```bash
  $ docker run -it  --privileged=true -v /mydocker/u:/tmp --name u1 ubuntu
  ```

- 容器 2 继承容器 1 的卷规则

  ```bash
  $ ·	docker run -it  --privileged=true --volumes-from u1 --name u2 ubuntu
  ```

