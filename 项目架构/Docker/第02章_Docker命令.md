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

### 2.1 docker images [option]

列出本地主机上的镜像。

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

### 2.2 docker search [option] [image name]

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

### 2.3 docker pull [image name]

下载镜像，可以带上 TAG 或者不带：

- `docker pull imageName:TAG`

- `docker pull imageName`，等同于`docker pull imageName:latest`

### 2.4 docker system df

查看镜像、容器、数据卷所占用的空间。

### 2.5 docker rmi [image name/image id]

删除镜像。

- 删除单个：`docker rmi imageId`

  ```bash
  [root@myServer1 ~]# docker rmi -f hello-world
  Untagged: hello-world:latest
  Untagged: hello-world@sha256:94ebc7edf3401f299cd3376a1669bc0a49aef92d6d2669005f9bc5ef028dc333
  Deleted: sha256:feb5d9fea6a5e9606aa995e879d862b825965ba48de054caab5ef356dc6b3412
  ```

- 删除多个：`docker rmi -f imageName:TAG imageName:TAG ...`

- 删除全部：`docker rmi -f ${docker images -qa}`

### 面试题：docker 虚悬镜像

仓库名、标签都是\<none\>的镜像，俗称虚悬镜像 dangling image。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230109225918706-fe135b483329b2e8cd7c21bdca45c136-9e2697.png" alt="image-20230109225918706" style="zoom:80%;" />

## 3.容器命令

有镜像才能创建容器， 这是根本前提。