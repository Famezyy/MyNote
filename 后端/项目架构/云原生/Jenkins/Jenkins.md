# Jenkins

Jenkins 是一款自动化拉取、集成、构建、测试的管理系统。

## 1.安装部署

首先创建三台虚拟机，分别作为 Gitlab 服务器、测试服务器和 Jenkins 服务器。

### 1.1 安装Gitlab

官方网站：https://about.gitlab.com

中文安装文档：https://gitlab.cn/install/

安装至少需要 6 G内存。

#### 1.在SSH下安装

**安装依赖**

```bash
sudo yum install -y curl policycoreutils-python openssh-server perl
sudo systemctl enable sshd
sudo systemctl start sshd
```

**配置镜像**

```bash
curl -fsSL https://packages.gitlab.cn/repository/raw/scripts/setup.sh | /bin/bash
```

**开始安装**

```bash
sudo EXTERNAL_URL = "http://192.168.11.100" yum install -y gitlab-jh
```

除非指定密码，否则会生成一个默认密码并存储在`/etc/gitlab/initial_root_password`文件夹（会在 24 小时后被`gitlab-ctl reconfigure`自动删除，因此建议初始登录成功后立即修改初始密码 -- 在管理员选项处更改密码）。使用用户名`root`和密码登陆。

#### 2.在Docker下安装

**设置卷位置**

在设置其他所有内容之前，请配置一个新的环境变量 `$GITLAB_HOME`，指向配置、日志和数据文件所在的目录。 确保该目录存在并且已授予适当的权限。

对于 Linux 用户，将路径设置为 `/srv/gitlab`

```
export GITLAB_HOME=/srv/gitlab
```

极狐 GitLab 容器使用主机装载的卷来存储持久数据

| 本地位置              | 容器位置          | 使用                        |
| :-------------------- | :---------------- | :-------------------------- |
| `$GITLAB_HOME/data`   | `/var/opt/gitlab` | 用于存储应用程序数据        |
| `$GITLAB_HOME/logs`   | `/var/log/gitlab` | 用于存储日志                |
| `$GITLAB_HOME/config` | `/etc/gitlab`     | 用于存储极狐GitLab 配置文件 |

创建一个 `docker-compose.yml` 文件

```yaml
version: '3.6'
services:
  web:
    image: 'registry.gitlab.cn/omnibus/gitlab-jh:latest'
    restart: always
    hostname: 'gitlab'
    ports:
      - '80:80'
      - '443:443'
      - '22:22' # 注意不要端口冲突，22 端口用于 SSH 连接
    volumes:
      - '$GITLAB_HOME/config:/etc/gitlab'
      - '$GITLAB_HOME/logs:/var/log/gitlab'
      - '$GITLAB_HOME/data:/var/opt/gitlab'
    shm_size: '256m'
```

确保您在与 `docker-compose.yml` 相同的目录下并启动极狐 GitLab

```
docker compose up -d
```

#### 3.常用命令

```bash
# 启动所有组件
gitlab-ctl start
# 停止所有的组件
gitlab-ctl stop
# 重启所有 gitlab 组件
gitlab-ctl restart
# 查看服务状态
gitlab-ctl statue
# 启动服务
gitlab-ctl reconfigure
# 修改默认的配置文件
vi /etc/gitlab/gitlab.rb
# 查看日志
gitlab-ctl tail
```

#### 4.新建项目

新建空白项目 - 填写项目名称、提交 URL - 新建

### 1.2 安装Jenkins

中文文档：https://www.jenkins.io/zh/doc/book/installing/

#### 1.利用yum安装

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum upgrade
# Add required dependencies for the jenkins package
sudo yum install java-11-openjdk
sudo yum install jenkins
sudo systemctl daemon-reload
```

>   **提示**
>
>   `sudo update-alternatives --config java`可以用来切换默认 java 环境到 11

启动 Jenkins

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

> **授予 root 权限**
>
> ```bash
> gpasswd -a root jenkins
> vim /etc/default/jenkins
> # 添加以下两行
> JENKINS_USER="root"
> JENKINS_GROUP="root"
> ```

#### 2.利用war包安装

-   安装 JDK 运行环境

    ```bash
    sudo yum upgrade
    sudo yum install java-11-openjdk
    ```

-   下载 war 包并上传至服务器

-   执行 war 包：`java -jar jenkins.war`

#### 3.安装Maven

https://maven.apache.org/download.cgi

下载 Binary tar.gz 包并上传至服务器

解压并执行：`tar -zxvf apachi-maven-x.x.x-bin.tar.gz `

复制到其他地方：`mv apachi-maven-x.x.x /usr/local/maven`

>   **修改阿里云镜像**
>
>   打开`settings.xml`，修改一下地方
>
>   ```xml
>   <mirror>
>     <id>alimaven</id>
>     <mirrorOf>central</mirrorOf>
>     <name>aliyun maven</name>
>     <url>http://maven.aliyun.com/nexus/content/repositories/central/</url>
>   </mirror>
>   ```
>
>   同时可以在配置文件中修改下本次仓库地址：
>
>   ```xml
>   <localRepository>D:\Maven\repository</localRepository>
>   ```
>
>   maven 全局 jdk 版本：
>
>   ```xml
>   <profile>
>     <id>jdk-1.8</id>
>     <activation>
>       <activeByDefault>true</activeByDefault>
>       <jdk>1.8</jdk>
>     </activation>
>     <properties>
>       <maven.compiler.source>1.8</maven.compiler.source>
>       <maven.compiler.target>1.8</maven.compiler.target>
>       <maven.compiler.compilerVersion>1.8</maven.compiler.compilerVersion>
>     </properties>
>   </profile>
>   ```

#### 4.安装Git

```bash
yum install -y git
```

#### 5.访问

访问`localhost:8080`，安装推荐的插件

重启

## 2.快速开始

### 2.1 打包Maven项目

Manage Jenkins - Manage Plugins - Available Plugins - 选择 Maven Integration - install without restart

回到主页 - 新建 Item - 选择 Maven

编辑`Configure`：

-   `源码管理` 

    -   勾选 git - 输入仓库 URL 确实是否正常连接（需要提前在 Jenkins 主机上安装 git）

        >   **配置SSH**
        >
        >   -   在 Jenkins 服务器上切换到 Jenkins 用户：`sudo su -s /bin/bash jenkins`
        >   -   生成 SSH key：`ssh-keygen -o`
        >   -   访问 Gitlab 服务器：`ssh root@192.168.11.100`

    -   指定要拉取的分支：`main`

- `Pre Steps`

  指定前置处理，可以用来做些前置清理

  - 选择服务器

  - 编写一个脚本用来清除之前启动的服务`clean.sh`，放在项目的根目录下（同`pom.xml`）

    ```bash
    #!/bin/bash
    
    # 删除历史数据
    rm -rf ~/microservice
    
    # 以下代码会获取传入的参数
    appname=$1
    if [ -z $appname ];
    	then
    		exit
    fi
    # 然后找到启动的信息中带该参数的 java 程序并得到第二个字符串（pid）
    pid=`ps -ef | grep $appname | grep 'java -jar' | awk '{printf $2}'`
    
    # 使用 -z 做空值判断
    if [ -z $pid ];
    	then
    		echo "$appname not started"
    		exit
    	else
    		kill -9 $pid
    		echo "$appname stoping..."
    fi
    
    check=`ps -ef | grep $pid | grep java | awk '{printf $1}'`
    if [ -z $check ];
    	then
    		echo "$appname pid:$pid stopped"
    	else
    		echo "$appname stop failed"
    fi
    ```

  - `transition source`：`clean.sh`

  - `Remote directory`：`/tmp`(会放在`root/tmp/`）

  - `Exec command`：`bash /root/tmp/clean.sh SNAPSHOT`

-   `Build`

    指定正确的 POM 文件地址：`pom.xml`（放在`src`同目录下的情况）

    >   **配置SSH**
    >
    >   -   在 Jenkins 服务器上切换到 Jenkins 用户：`sudo su -s /bin/bash jenkins`
    >   -   生成 SSH key：`ssh-keygen -o`
    >   -   访问 Gitlab 服务器：`ssh root@192.168.11.100`

回到 Dashboard 主页，点击运行

观察控制台输出：构建执行状态 - 选择相应的任务

最后 JAR 包会生成在`/var/lib/jenkins/workspace/jenkins-maven/target/jenkins-test-0.0.1-SNAPSHOT.jar`

### 2.2 将JAR包发送到测试服务器

安装插件：`Publish Over SSH`

在`Configure System`中添加测试服务器：

-   最下面找到`Publish over SSH`
-   新增`SSH Servers`
-   填写`Name`、`Hostname`、`Username`
-   选择高级选项`Use password authentication`
-   填写密码
-   `Test Configuration`测试是否能连接上

在上一节的基础上继续编辑`Configure`：

-   `Post Steps`

    -   选择`Send files or execute commandfs over SSH`
    -   选择之前配好的`SSH Server`
    -   指定`Source file`，例如：`**/*SNAPSHOT.jar`
    -   指定`Remove prefix`，例如：`/target`
    -   指定`Remote directory`，例如：`/microservice`（默认放在`~`下，这条命令相当于放在`/root/microservice`下）

    >   按照以上的配置，最终 JAR 会被放在`/root/microservice`中

-   `Exec command`

    指定执行的命令，例如：`nohup java -jar /root/microservice/*SNAPSHOT.jar > /root/microservice/mylog.log 2>&1 &`

    （不要指定阻塞命令，否则 Jinkens 最后执行完后会一直阻塞直到超时）

    >   **数据流重定向**
    >
    >   数据流重定向就是将某个命令执行后应该要出现在屏幕上的数据传输到其他地方
    >
    >   -   标准输入（stdin）：代码为 0，使用`<`或`<<`
    >   -   标准输出（stdout）：代码为 1，使用`>`或`>>`
    >   -   标准错误输出（stderr）：代码为 2，使用`2>`或`2>>`
    >   -   `>`：覆盖写
    >   -   `>>`：追加写

回到 Dashboard 主页，点击运行

## 3.构建触发器

1. 当 POM 依赖构建时重新构建此项目
2. 触发远程构建
3. 其他工程构建后触发构建
4. 定时构建（使用 cron 表达式）
5. Github 远程构建
6. 轮询 SSM，当前代码发生变化则构建（使用 cron 表达式）

### 3.1 触发远程构建（不推荐）

#### 1.配置Jenkins

- 触发构建器 - 触发远程构建 - 输入自定义令牌（例如：`123123`）

- 下载插件`Build Authorization Token Root`，用于免除 Jenkins 的权限认证

  > **补充**
  >
  > 直接将钩子 URL 直接注册到 Gitlab 会因为没有权限而无法调用。
  >
  > 使用时使用插件提供的端点：`buildByToken/build?job=NAME&token=SECRET`，用 POSTMAN 向`http://192.168.11.101:8080/buildByToken/build?job=jenkins-maven&token=123123`发送请求就会触发项目构建。
  >
  > 参考：https://plugins.jenkins.io/build-token-root/

#### 2.配置Gitlab

- 项目 - 设置 - Webhooks
- 填写钩子 URL：`http://192.168.11.101:8080/buildByToken/build?job=jenkins-maven&token=123123`
- 选择`合并请求事件`
- 取消`SSL`请求

> **注意**
>
> 如果是本地请求，Gitlab 默认会限制：`Url is blocked: Requests to the local network are not allowed`
>
> 需要切换到管理员身份，设置 - 网络 - 出战请求 - 勾选`允许来自 web hooks 和服务对本地网络的请求` - 保存

但是这种方式在合并的时候**会触发两次构建**：第一次是提交合并请求；第二次是通过合并请求

### 3.2 定时触发

#### 1.cron表达式

标准 cron：https://crontab.guru

```bash
 ┌───────────── minute (0 - 59)
 │ ┌───────────── hour (0 - 23)
 │ │ ┌───────────── day of month (1 - 31)
 │ │ │ ┌───────────── month (1 - 12)
 │ │ │ │ ┌───────────── day of week (0 - 6) (Sunday to Saturday;
 │ │ │ │ │                                       7 is also Sunday on some systems)
 │ │ │ │ │
 │ │ │ │ │
 * * * * *  schedule command to execute
```

Jenkins 定义了一种高级计算方式：

如果是`*/10`，则表示每个`10`分钟，如果定义了`H/10`，则会首先对项目名称取 hash 值作为起始分钟，间隔`10`分钟，这样有利于散列各个项目的构建时间

还可以定义范围：对于`H(1-30)`则表示从`1~30`中随机取一个数字做为间隔，这个间隔一旦取出就不会变

例子：

- `H * * * * `：每个小时的具体某个散列时间
- `*/10 * * * *`：每隔 10 分钟
- `H/10 * * * *`：得到具体某个散列时间，如 56，每次间隔 10 分钟，则下次执行在 6 分的时候

- `H(1-30) 2 * * 1-6`：周一至周六，每天的凌晨 2 点的 24（随机）分执行
- `H(1-30) 2-5 * * 1-6`：周一至周六，每天的凌晨 2 点 25（随机），3 点 25 分，4 点 25 分，5 点 25 分执行
- `H(1-30) 0-6/2 * * 1-6`：周一至周六，每天的凌晨 0 点 13（随机），2 点 13 分，4 点 13 分，6 点 13 分执行

#### 2.轮询 SCM

指定`* * * * *`，每分钟检查一次对应的 branch，不更新代码时不会触发构建，一旦有代码更新，则会触发构建

## 4.配置邮箱

- 注册有效的 SMTP 服务器
- Jenkins - `系统配置`
  - 确认`Jenkins URL`，注意 Jenkins URL 的可访问性（邮件中有打开 Jenkins 的链接）
  - 填写`系统管理员邮件地址` - 与 SMTP 服务器对应的发件地址
  - 配置`Extended E-mail Notification`
    - 填写 SMTP 服务器
    - 添加`Credential`
      - 使用`Username with password`类型
      - 填写`用户名`和`密码`（SMTP 登陆凭证）
    - 确认邮件格式`Default Content`
    - 确认`Default Triggers`
  - 配置`邮件通知`
    - 填写 SMTP 服务器
    - 高级 - 使用 SMTP 认证，填写用户名和密码
  - 确认项目的邮件配置
    - 打开`构建后操作` - `Editable Email Notification` - `Advanced setting` - `Triggers`
    - 确认用户组是否正确

## 5.容器化构建

三种方式：

1. **外挂目录**

   将 JAR 包放在宿主机，通过容器卷关联 Docker 容器的目录，Jekins 只需要把 JAR 包放在容器卷下，通过运行 Docker 指令启动服务。

2. **把 JAR 包打包到容器**

   Jekins 将 JAR 包和 dockerfile 一同发送到宿主机，运行并生成镜像，通过镜像启动服务。

3. **k8s 云原生**

   Jekins 将 JAR 包和 dockerfile 一同发送到处理服务器，运行并生成镜像，由服务器将镜像推送到镜像仓库`Harobor`，然后由 k8s 拉取该镜像并创建容器服务。

### 5.1 外挂目录

- `pre steps`

  - 在`pom`同目录下编写`clean.sh`，查找是否存在 JDK8 镜像，清除`/root/microservice`目录，并停掉运行的容器

    ```bash
    #!/bin/bash
    
    SERVICEDIR=/root/workspace/microservice
    containername=$1
    
    # 安装镜像
    image=`docker images -f reference=openjdk:8 | grep openjdk`
    if [[ -z $image ]];then
    echo "=====openjdk:8镜像不存在====="
    echo "=====开始下载镜像====="
    docker pull openjdk:8
    else
    echo "=====openjdk:8镜像已存在====="
    fi
    
    # 清除目录
    rm -rf $SERVICEDIR
    
    if [ -z $containername ];then
    exit
    fi
    
    # 清除 container
    container=`docker ps -af name=$containername | grep $containername`
    if [[ -z $container ]];then
    exit
    else
    echo "=====开始清除$containername====="
    docker rm -f $containername
    fi
    ```

  - 发送`clean.sh`

    - SSH server
    - Source file：`clean.sh`
    - Remote directory：`/workspace/config`（会默认置于`~`）
    - Exec command：`bash /root/workspace/config/clean.sh [containername]`

- `post steps`

  发送 JAR 包到容器的`/root/workspace/microservice`目录

  - SSH Server
  - Source file：`**/*SNAPSHOT.jar`
  - Remove prefix：`/target`
  - Remote directory：`/workspace/microservice`

  执行`rm -rf /root/workspace/config && docker run -d -p 8080:8080 --name [containername] -v /root/workspace/microservice/xxx.jar:/root/workspace/microservice/xxx.jar openjdk:8 java -jar /root/workspace/microservice/xxx.jar`

### 5.2 打包到容器

- `pre steps`

  - 在`pom`同目录下编写`clean.sh`

    - 查找是否存在 JDK8 镜像，没有则拉取

    - 查看是否存在微服务镜像，有则删除

    - 清除`/root/workspace/[image_id]`目录

    - 删除运行的容器

    ```bash
    #!/bin/bash
    
    containername=$1
    jarfilename=$2
    IMAGE_NAME=$3
    IMAGE_TAG=$4
    SERVICEDIR=/root/workspace/${IMAGE_NAME}_${TAG}
    
    # 检查 JDK8 镜像
    jdk_image=`docker images -f reference=openjdk:8 | grep openjdk`
    if [[ -z $jdk_image ]];then
        echo "=====openjdk:8镜像不存在====="
        echo "=====开始下载镜像====="
        docker pull openjdk:8
    else
        echo "=====openjdk:8镜像已存在====="
    fi
    
    # 检查微服务镜像
    service_image=`docker images -f reference=${IMAGE_NAME}:${TAG} | grep ${IMAGE_NAME}:${TAG}`
    if [[ ! -z $service_image ]];then
        echo "====删除已有微服务镜像===="
        docker rmi ${IMAGE_NAME}:${TAG}
        # tag=`docker images -f reference=${iamge_id} | grep ${iamge_id} | awk 'pringf $2'`
        # docker rmi microservice:$tag
    fi
    
    # 清除目录
    rm -rf $SERVICEDIR
    
    # 创建 Dockerfile
    echo "=====创建dockerfile====="
    mkdir $SERVICEDIR
    cat > $SERVICEDIR/Dockerfile << EOF
    FROM openjdk:8
    WORKDIR ${SERVICEDIR}
    ADD $jarfilename.jar ${SERVICEDIR}/$jarfilename.jar
    EXPOSE 8080
    ENTRYPOINT ["java", "-jar", "${SERVICEDIR}/$jarfilename.jar"]
    EOF
    
    # 删除微服务容器
    container=`docker ps -f name=$containername | grep $containername`
    if [[ -z $container ]];then
    exit
    else
    echo "=====开始清除$containername====="
    docker rm -f $containername
    fi
    ```

  - 发送`clean.sh`

    - SSH server
    - Source file：`clean.sh`
    - Remote directory：`/workspace/[image_name]:[image_tag]/config`（会默认置于`~`）
    - Exec command：`bash /root/workspace/[image_name]:[image_tag]/config/clean.sh [containername] [jarfilename] [image_name] [image_tag]` 

- `post steps`

  发送 JAR 包到应用服务器的`/workspace/[image_name]:[image_tag]`目录

  - SSH Server
  - Source file：`**/*SNAPSHOT.jar`
  - Remove prefix：`/target`
  - Remote directory：`/workspace/[image_name]:[image_tag]`

  执行

  ```bash
  image_id=[image_id]
  container_name=[container_name]
  IMAGE_NAME=[image_name]
  TAG=[tag]
  docker build -t ${IMAGE_NAME}:${TAG} /root/workspace/${IMAGE_NAME}_${TAG}/
  docker run -d -p 8080:8080 --name ${container_name} ${IMAGE_NAME}_${TAG}
  rm -rf /workspace/image_name:image_tag/config
  ```

### 5.3 Harbor管理镜像

- 在 Harbor 和应用服务器上建立 SSH 连接

- 推送到 Harbor 仓库构建镜像，步骤同打包到容器（或者可以在 Jenkins 服务器上构建镜像然后推送到 Harbor 上，此时使用`执行 shell`选项）

  - `pre steps`发送`clean.sh`

    ```bash
    #!/bin/bash
    
    containername=$1
    jarfilename=$2
    IMAGE_NAME=$3
    IMAGE_TAG=$4
    PROJECT_NAME=$5
    # 目标服务器的 JAR 地址
    SERVICEDIR=/root/workspace/${IMAGE_NAME}_${TAG}
    # Jenkins 本地的 JAR 地址，可以在本地docker构建好后再推送到 Harbor
    # JAR_DIR=/var/lib/jenkins/workspace/${PROJECT_NAME}/target
    
    # 检查 JDK8 镜像
    jdk_image=`docker images -f reference=openjdk:8 | grep openjdk`
    if [[ -z $jdk_image ]];then
        echo "=====openjdk:8镜像不存在====="
        echo "=====开始下载镜像====="
        docker pull openjdk:8
    else
        echo "=====openjdk:8镜像已存在====="
    fi
    
    # 检查微服务镜像
    service_image=`docker images -f reference=${IMAGE_NAME}:${TAG} | grep ${IMAGE_NAME}:${TAG}`
    if [[ ! -z $service_image ]];then
        echo "====删除已有微服务镜像===="
        docker rmi ${IMAGE_NAME}:${TAG}
        # tag=`docker images -f reference=${iamge_id} | grep ${iamge_id} | awk 'pringf $2'`
        # docker rmi microservice:$tag
    fi
    
    # 清除目录
    rm -rf $SERVICEDIR
    
    # 创建 Dockerfile
    echo "=====创建dockerfile====="
    mkdir $SERVICEDIR
    cat > $SERVICEDIR/Dockerfile << EOF
    FROM openjdk:8
    WORKDIR ${SERVICEDIR}
    ADD $jarfilename.jar ${SERVICEDIR}/$jarfilename.jar
    EXPOSE 8080
    ENTRYPOINT ["java", "-jar", "${SERVICEDIR}/$jarfilename.jar"]
    EOF
    ```

    ```bash
    bash /root/workspace/[image_name]:[image_tag]/config/clean.sh [containername] [jarfilename] [image_name] [image_tag] [project_name]
    ```
  
  - `post steps`
  
    发送 JAR 包到 Harbor 服务器的`/workspace/[image_name]:[image_tag]`目录
  
    在 Harbor 服务器执行`build`
  
    ```bash
    HARBOR_USER=admin
    HARBOR_PASSWORD=Harbor12345
    HARBOR_IP=192.168.11.100:80
    IMAGE_NAME=[image_name]
    TAG=[tag]
    docker login -u ${HARBOR_USER} -p HARBOR_PASSWORD ${HARBOR_IP}
    docker build -t ${HARBOR_IP}/${IMAGE_NAME}:${TAG} .
    docker push ${HARBOR_IP}/
    rm -rf /root/workspace/config
    ```
  
    在应用服务器执行拉取和运行
  
    ```bash
    HARBOR_IP=192.168.11.100:80
    IMAGE_NAME=[image_name]
    TAG=[tag]
    containername=[containername]
    
    # 检查微服务镜像
    service_image=`docker images -f reference=${IMAGE_NAME}:${TAG} | grep ${IMAGE_NAME}:${TAG}`
    if [[ ! -z $service_image ]];then
        echo "====删除已有微服务镜像===="
        docker rmi ${IMAGE_NAME}:${TAG}
        # tag=`docker images -f reference=${iamge_id} | grep ${iamge_id} | awk 'pringf $2'`
        # docker rmi microservice:$tag
    fi
    
    # 删除微服务容器
    container=`docker ps -f name=$containername | grep $containername`
    if [[ -z $container ]];then
    exit
    else
    echo "=====开始清除$containername====="
    docker rm -f $containername
    fi
    
    docker pull ${HARBOR_IP}/${IMAGE_NAME}:${tag}
    docker run -d -p 8080:8080 --name ${container_name} ${IMAGE_NAME}:${TAG}
    ```

## 6.Jenkins集群

**添加节点**

- 确保节点环境配置了 Java

- 打开主机点的“系统管理” - “节点管理” - “新建节点”
-  填写“节点名称”，选择“固定”
- 配置节点
  - "远程工作目录"：`/root`
  - “Number of executors”：可并发执行数
  - “标签”：与”名称“一样
  - “用法”：尽可能地使用
  - “启动方式”：通过 SSH
    - 填写主机地址
    - 添加认证方式
    - 选择"Non verifying verification strategy"

**配置 JOB 可并发执行**

- “general”：选择“在必要的时候并发构建”

  > **注意**
  >
  > 也可以选择“限制项目的运行节点”。

## 7.流水线Pipeline

“新建项目” - 选择“流水线”。

**完整语法**

- pipeline：整条流水线
- agent：指定在哪台节点执行
- stages：所有阶段
- state：某一阶段
- steps：阶段内的每一步

**hello-world**

```bash
pipeline {
    agent any

    stages {
        stage('拉取代码') {
            steps {
                echo '拉取成功'
            }
        }
        stage('执行构建') {
            steps {
                echo '构建完成'
            }
        }
        stage('发送JAR包') {
            steps {
                echo '构建完成'
            }
        }
    }
}
```

**安装 blue ocean UI 插件**

更加丰富的 pipeline 执行过程可视化插件。

**案例：自动打包Docker镜像外挂JAR包**

可利用“流水线语法生成代码”，例如 git 的拉取操作、“send build artifacts over SSH”等。

```bash
pipeline {
    agent any
    
    tools {
    	// 与全局工具设置的 maven 版本一致
    	maven "maven4"
    }

    stages {
        stage('拉取代码') {
            steps {
            	// 	默认是拉取到 /root/workspace/[pipeline名称]/
            	git branch: 'main', url: 'git@192.168.11.100:gitlab-instance-568d7044/jenkins-test.git'
                echo '拉取成功'
            }
        }
        stage('执行构建') {
            steps {
            	// sh "mvn --version"
            	sh "mvn clean package"
                echo '构建完成'
            }
        }
        stage('配置docker运行环境') {
        	steps {
        		echo '执行clean'
        		sshPublisher(publishers: [sshPublisherDesc(configName: 'test-server', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: 'bash /root/workspace/config/clean.sh microservice', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '/workspace/config', remoteDirectorySDF: false, removePrefix: '', sourceFiles: 'clean.sh')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
        	}
        }
        stage('发送JAR包') {
            steps {
   				sshPublisher(publishers: [sshPublisherDesc(configName: 'test-server', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: 'rm -rf /root/workspace/config && docker run -d -p 8080:8080 --name microservice -v /root/workspace/microservice/jenkins-test-0.0.1-SNAPSHOT.jar:/root/workspace/microservice/jenkins-test-0.0.1-SNAPSHOT.jar openjdk:8 java -jar /root/workspace/microservice/jenkins-test-0.0.1-SNAPSHOT.jar', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '/workspace/microservice', remoteDirectorySDF: false, removePrefix: '/target', sourceFiles: '**/jenkins-test-0.0.1-SNAPSHOT.jar')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
                echo '启动完成'
            }
        }
    }
}
```

## 8.多分支JOB

对不同的分支（main、test）执行不同的构建最终运行在不同的服务器上。

- “新建任务” - “多分支流水线”
- 填写“名称”
- 填写“Git仓库”
- 设置“扫描多分支流水线触发器”，设置处罚间隔

它依赖于各个分支下的`Jenkinsfile`（流水线Pipeline定义文件），在不同分支下创建不同的 pipeline 文件，起名`Jenkinsfile`。

