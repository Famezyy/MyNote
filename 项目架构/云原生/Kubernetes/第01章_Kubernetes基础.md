# 第01章_Kubernetes基础

## 1.基本介绍

### 1.1 使用背景

容器化部署方式给带来很多的便利，但是也会出现一些问题，比如说：

- 一个容器故障停机了，怎么样让另外一个容器立刻启动去替补停机的容器
- 当并发访问量变大的时候，怎么样做到横向扩展容器数量

这些容器管理的问题统称为**容器编排**问题，为了解决这些容器编排问题，就产生了一些容器编排的软件：

- **Swarm**：Docker 自己的容器编排工具
- **Mesos**：Apache 的一个资源统一管控的工具，需要和 Marathon 结合使用
- **Kubernetes**：Google 开源的的容器编排工具

### 1.2 简介

Kubernetes 是一个全新的基于容器技术的分布式架构领先方案，是谷歌严格保密十几年的秘密武器 - Borg 系统的一个开源版本，于2014年9月发布第一个版本，2015年7月发布第一个正式版本。

Kubernetes 的本质是**一组服务器集群**，它可以在集群的每个节点上运行特定的程序，来对节点中的容器进行管理。目的是实现资源管理的自动化，主要提供了如下的主要功能：

- **自我修复**：一旦某一个容器崩溃，能够在1秒中左右迅速启动新的容器
- **弹性伸缩**：可以根据需要，自动对集群中正在运行的容器数量进行调整
- **服务发现**：服务可以通过自动发现的形式找到它所依赖的服务
- **负载均衡**：如果一个服务起动了多个容器，能够自动实现请求的负载均衡
- **版本回退**：如果发现新发布的程序版本有问题，可以立即回退到原来的版本
- **存储编排**：可以根据容器自身的需求自动创建存储卷

### 1.3 组件

一个 Kubernetes 集群主要是由**控制节点（master）**、**工作节点（node）**构成，每个节点上都会安装不同的组件。

**master**：集群的控制平面，负责集群的决策 ( 管理 )

- **ApiServer** : 资源操作的唯一入口，接收用户输入的命令，提供认证、授权、API 注册和发现等机制

- **Scheduler** : 负责集群资源调度，按照预定的调度策略将 Pod 调度到相应的 node 节点上

- **ControllerManager** : 负责维护集群的状态，比如程序部署安排、故障检测、自动扩展、滚动更新等

- **Etcd** ：负责存储集群中各种资源对象的信息

**node**：集群的数据平面，负责为容器提供运行环境 ( 干活 )

- **Kubelet** : 负责维护容器的生命周期，即通过控制 docker，来创建、更新、销毁容器
- **KubeProxy** : 负责提供集群内部的服务发现和负载均衡
- **Docker** : 负责节点上容器的各种操作

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202301281701331.png" alt="image-20200406184656917" style="zoom:80%;" />

下面，以部署一个 nginx 服务来说明 Kubernetes 系统各个组件调用关系：

1. 首先要明确，一旦 Kubernetes 环境启动之后，master 和 node 都会将自身的信息存储到`etcd`数据库中
2. 一个 nginx 服务的安装请求会首先被发送到 master 节点的`ApiServer`组件
3. `ApiServer`组件会调用`Scheduler`组件来决定到底应该把这个服务安装到哪个 node 节点上，此时，它会从`etcd`中读取各个 node 节点的信息，然后按照一定的算法进行选择，并将结果告知`ApiServer`
4. `ApiServer`调用`controller-manager`去调度 Node 节点安装 nginx 服务
5. `kubelet`接收到指令后，会通知`docker`，然后由`docker`来启动一个 nginx 的`pod`

至此，一个 nginx 服务就运行了，如果需要访问 nginx，就需要通过`kube-proxy`来对`pod`产生访问的代理。

### 1.4 相关概念

**Master**：集群控制节点，每个集群需要至少一个 master 节点负责集群的管控

**Node**：工作负载节点，由 master 分配容器到这些 node 工作节点上，然后 node 节点上的 docker 负责容器的运行

**Pod**：kubernetes 的最小控制单元，容器都是运行在 pod 中的，一个 pod 中可以有 1 个或者多个容器

**Controller**：控制器，通过它来实现对 pod 的管理，比如启动 pod、停止 pod、伸缩 pod 的数量等等

**Service**：pod 对外服务的统一入口，下面可以维护者同一类的多个 pod

**Label**：标签，用于对 pod 进行分类，同一类 pod 会拥有相同的标签

**NameSpace**：命名空间，用来隔离 pod 的运行环境

## 2.集群环境搭建

目前生产部署 Kubernetes 单点场景使用**minikuber**，而部署集群时主要有两种方式：

**kubeadm**

Kubeadm 是一个K8s 部署工具，提供kubeadm init 和kubeadm join，用于快速部署Kubernetes 集群。

官方地址：https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/

**二进制包**

从 github 下载发行版的二进制包，手动部署每个组件，组成 Kubernetes 集群。

Kubeadm 降低部署门槛，但屏蔽了很多细节，遇到问题很难排查。如果想更容易可控，推荐使用二进制包部署Kubernetes 集群，虽然手动部署麻烦点，期间可以学习很多工作原理，也利于后期维护。

### 2.4 准备环境

本次环境搭建使用一主二从，因此提前创建好 3 台虚拟机。

| 角色   | IP地址        | 所需组件                          |
| ------ | ------------- | --------------------------------- |
| master | 172.16.19.200 | docker，kubectl，kubeadm，kubelet |
| node01 | 172.16.19.201 | docker，kubectl，kubeadm，kubelet |
| node02 | 172.16.19.202 | docker，kubectl，kubeadm，kubelet |

### 2.5 环境初始化

#### 1.检查操作系统的版本

```bash
# 此方式下安装 Kubernetes 集群要求 Centos 版本要在 7.5 或之上
cat /etc/redhat-release
```

#### 2.主机名解析

为了方便集群节点间的直接调用，在这个配置一下主机名解析，企业中推荐使用内部 DNS 服务器

```bash
# 主机名解析，编辑三台服务器的 /etc/hosts 文件，添加下面内容
vim /etc/hosts
172.16.19.200 master
172.16.19.201 node1
172.16.19.202 node2
```

#### 3.时间同步

kubernetes要求集群中的节点时间必须精确一直，这里使用chronyd服务从网络同步时间

企业中建议配置内部的会见同步服务器

```bash
# 启动 chronyd 服务
systemctl start chronyd
systemctl enable chronyd
date
```

#### 4.禁用iptable和firewalld服务

kubernetes 和 docker 在运行的中会产生大量的 iptables 规则，为了不让系统规则跟它们混淆，直接关闭系统的规则

```bash
# 关闭 firewalld 服务
systemctl stop firewalld
systemctl disable firewalld
# 关闭 iptables 服务
systemctl stop iptables
systemctl disable iptables
```

#### 2.6.5 禁用selinux

selinux 是 linux 系统下的一个安全服务，如果不关闭它，在安装集群中会产生各种各样的奇葩问题

查看 selinux 状态

```bash
getenforce
```

禁用

```bash
# 编辑 /etc/selinux/config 文件，修改 SELINUX 的值为 disable
# 注意修改完毕之后需要重启 linux 服务
vim /etc/selinux/config
SELINUX=disabled
```

#### 2.6.6 禁用swap分区

swap 分区指的是虚拟内存分区，它的作用是物理内存使用完，之后将磁盘空间虚拟成内存来使用，启用 swap 设备会对系统的性能产生非常负面的影响，因此 kubernetes 要求每个节点都要禁用 swap 设备，但是如果因为某些原因确实不能关闭 swap 分区，就需要在集群安装过程中通过明确的参数进行配置说明

查看 swap 分区状态

```bash
free -m
```

禁用

```bash
# 编辑分区配置文件 /etc/fstab，注释掉 swap 分区一行
# 注意修改完毕之后需要重启 linux 服务
vim /etc/fstab
# /dev/mapper/centos-swap swap
```

#### 2.6.7 修改linux的内核参数

```bash
# 修改 linux 的内核采纳数，添加网桥过滤和地址转发功能
# 编辑 /etc/sysctl.d/k8s.conf 文件，添加如下配置：
vim /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1

# 重新加载配置
sysctl -p
# 加载网桥过滤模块
modprobe br_netfilter
# 查看网桥过滤模块是否加载成功
lsmod | grep br_netfilter
```

#### 2.6.8 配置ipvs功能

在 kubernetes 中 service 有两种带来模型，一种是基于 iptables 的，一种是基于 ipvs 的两者比较的话，ipvs 的性能明显要高一些，但是如果要使用它，需要手动载入 ipvs 模块

```bash
# 1.安装 ipset 和 ipvsadm
yum install ipset ipvsadm -y
# 2.添加需要加载的模块写入脚本文件
cat <<EOF> /etc/sysconfig/modules/ipvs.modules
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
# 高版本内核下使用
# modprobe -- nf_conntrack
EOF
# 3.为脚本添加执行权限
chmod +x /etc/sysconfig/modules/ipvs.modules
# 4.执行脚本文件
/bin/bash /etc/sysconfig/modules/ipvs.modules
# 5.查看对应的模块是否加载成功
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

### 2.6 安装Docker

参考<a href="../Docker/第01章_Docker简介">Docker</a>章节。

```bash
# 添加一个配置文件
# Docker 在默认情况下使用 Vgroup Driver 为 cgroupfs，而 Kubernetes 推荐使用 systemd 来替代 cgroupfs
mkdir /etc/docker
cat <<EOF> /etc/docker/daemon.json
{
	"exec-opts": ["native.cgroupdriver=systemd"],
	"registry-mirrors": ["https://kn0t2bca.mirror.aliyuncs.com"]
}
EOF

# 5.启动dokcer
systemctl restart docker
systemctl enable docker
```

### 2.7 安装Kubernetes

阿里云：https://developer.aliyun.com/mirror/kubernetes?spm=a2c6h.13651102.0.0.73281b11YKT2nT

```bash
# 国内
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

```bash
# 国外
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

```bash
yum install -y kubelet kubeadm kubectl
```

```bash
# 配置 kubelet 的 cgroup
# 编辑 /etc/sysconfig/kubelet, 添加下面的配置
vim /etc/sysconfig/kubelet
KUBELET_CGROUP_ARGS="--cgroup-driver=systemd"
KUBE_PROXY_MODE="ipvs"
```

```bash
systemctl enable kubelet && systemctl start kubelet
```

### 2.8 配置Kubernetes集群

#### 1.准备集群镜像

在安装 kubernetes 集群之前，必须要提前准备好集群需要的镜像，所需镜像可以通过下面命令查看

```bash
kubeadm config images list
```

**下载镜像**

```bash
images=(
  registry.k8s.io/kube-apiserver:v1.26.1
  registry.k8s.io/kube-controller-manager:v1.26.1
  registry.k8s.io/kube-scheduler:v1.26.1
  registry.k8s.io/kube-proxy:v1.26.1
  registry.k8s.io/pause:3.9
  registry.k8s.io/etcd:3.5.6-0
  registry.k8s.io/coredns/coredns:v1.9.3
)
```

```bash
# 国内
for imageName in ${images[@]};do
	# 从阿里源拉取镜像文件，但是名字格式不同
	docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
	# 重命名镜像名
	docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
	docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName 
done
```

```bash
# 国外
for imageName in ${images[@]};do
	docker pull $imageName
done
```

#### 2.集群初始化

下面的操作只需要在 master 节点上执行即可

```bash
# 创建集群
kubeadm init --apiserver-advertise-address=192.168.11.100 --kubernetes-version=1.26.1 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16 # --image-repository=registry.aliyuncs.com/google_containers

# 执行下面的命令
export KUBECONFIG=/etc/kubernetes/admin.conf
```

> **提示**
>
> 当出现`E0205 12:44:31.717817    1505 memcache.go:238] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
> The connection to the server localhost:8080 was refused - did you specify the right host or port?`错误时，可尝试执行`export KUBECONFIG=/etc/kubernetes/admin.conf`。

> **问题**
>
> 出现以下错误时`[ERROR CRI]: container runtime is not running`的解决方案：
>
> ```bash
> rm -rf /etc/containerd/config.toml
> systemctl restart containerd
> ```

等初始化完成后会显示 node 节点加入集群的命令，只需要在 node 节点上执行即可。

> **重新获取加入命令**
>
> ```bash
> kubeadm token create --print-join-command
> ```

#### 3.安装网络插件

kubernetes 支持多种网络插件，如 flannel，calico，cannal 等，本次选择 flannel，只在 master 节点操作即可，插件会通过 DaemonSet 控制器同步安装

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

> **注意**
>
> 国内使用时修改文件中 quay.io 仓库为`quay-mirror.qiniu.com`。
>
> 由于外网不好访问，如果出现无法访问的情况，可以直接用下面的：
>
> ```bash
> https://github.com/flannel-io/flannel/tree/master/Documentation/kube-flannel.yml
> ```

使用配置文件启动 fannel

```bash
kubectl apply -f kube-flannel.yml
```

> **问题**
>
> - 若是集群状态一直是 notready，用下面语句查看原因
>
>   ```bash
>   journalctl -f -u kubelet.service
>   ```
>
> - 若显示`cni plugin not initialized`，可尝试重启`contained`

#### 4.重置

```bash
kubeadm reset -f
ipvsadm --clear
rm -rf /etc/cni/net.d
rm -rf $HOME/.kube/config
ip link delete flannel.1
ip link delete cni0
rm -rf /etc/containerd/config.toml
systemctl restart containerd
```

### 2.9 测试

**创建一个 nginx 服务**

```bash
kubectl create deployment nginx --image=nginx:1.14-alpine
```

**暴露端口**

```bash
kubectl expose deploy nginx --port=80 --target-port=80 --type=NodePort
```

**查看服务**

```bash
kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6db6dff665-nr59s   1/1     Running   0          9s

kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        81s
nginx        NodePort    10.106.114.184   <none>        80:32513/TCP   10s
```

> **问题**
>
> - pod 始终处于`containerCreating`状态时，使用以下命令查看日志
>
>   ```bash
>   kubectl describe pod nginx-6db6dff665-9fglb
>   ```
>
> - 错误为`"cni0" already has an IP address`时，重置 kubernetes

可以访问任意节点 IP：`192.168.11.101:32513`

## 3.资源管理

### 3.1 资源管理介绍

在 kubernetes 中，所有的内容都抽象为**资源**，用户需要通过操作资源来管理 kubernetes。

kubernetes 的本质上就是一个集群系统，用户可以在集群中部署各种服务，所谓的部署服务，其实就是在 kubernetes 集群中运行一个个的容器，并将指定的程序跑在容器中。

kubernetes 的最小管理单元是`pod`而不是容器，所以只能将容器放在`pod`中，而 kubernetes 一般也不会直接管理`pod`，而是通过`pod控制器`来管理`pod`的。

`pod`可以提供服务之后，就要考虑如何访问`pod`中服务，kubernetes 提供了`service`资源实现这个功能。

当然，如果`pod`中程序的数据需要持久化，kubernetes 还提供了各种`存储`系统。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200406225334627-102ffc00a84026635682f8cfd33e6d1e-ef403f.png" alt="img" style="zoom: 67%;" />

学习 kubernetes 的核心，就是学习如何对集群上的`Pod、Pod控制器、Service、存储`等各种资源进行操作。

### 3.2 YAML语言介绍

YAML 是一个类似 XML、JSON 的标记性语言。它强调以**数据**为中心，并不是以标识语言为重点。因而 YAML 本身的定义比较简单，号称"一种人性化的数据格式语言"。

YAML的语法比较简单，主要有下面几个：

- 大小写敏感
- 使用缩进表示层级关系
- 缩进不允许使用 tab，只允许空格（低版本限制）
- 缩进的空格数不重要，只要相同层级的元素左对齐即可
- '#'表示注释

YAML 支持以下几种数据类型：

- 纯量：单个的、不可再分的值
- 对象：键值对的集合，又称为映射（mapping）/ 哈希（hash） / 字典（dictionary）
- 数组：一组按次序排列的值，又称为序列（sequence） / 列表（list）

```yaml
# 纯量, 就是指的一个简单的值，字符串、布尔值、整数、浮点数、Null、时间、日期
# 1 布尔类型
c1: true (或者True)
# 2 整型
c2: 234
# 3 浮点型
c3: 3.14
# 4 null类型 
c4: ~  # 使用~表示null
# 5 日期类型
c5: 2018-02-17    # 日期必须使用 ISO 8601 格式，即 yyyy-MM-dd
# 6 时间类型
c6: 2018-02-17T15:02:31+08:00  # 时间使用 ISO 8601 格式，时间和日期之间使用T连接，最后使用 + 代表时区
# 7 字符串类型
c7: heima     # 简单写法，直接写值 , 如果字符串中间有特殊字符，必须使用双引号或者单引号包裹 
c8: line1
    line2     # 字符串过多的情况可以拆成多行，每一行会被转化成一个空格
# 对象
# 形式一（推荐）:
heima:
  age: 15
  address: Beijing
# 形式二（了解）:
heima: {age: 15, address: Beijing}
# 数组
# 形式一（推荐）:
address:
  - 顺义
  - 昌平  
# 形式二（了解）:
address: [顺义,昌平]
```

> 小提示：
>
> 1 书写yaml切记`:` 后面要加一个空格
>
> 2 如果需要将多段 yaml 配置放在一个文件中，中间要使用`---`分隔
>
> 3 下面是一个 yaml 转 json 的网站，可以通过它验证 yaml 是否书写正确
>
> https://www.json2yaml.com/convert-yaml-to-json

### 3.3 资源管理方式

- 命令式对象管理：直接使用命令去操作 kubernetes 资源

  ```bash
  kubectl run nginx-pod --image=nginx:1.17.1 --port=80
  ```

- 命令式对象配置：通过命令配置和配置文件去操作 kubernetes 资源

  ```bash
  kubectl create/patch -f nginx-pod.yaml
  ```

- 声明式对象配置：通过 apply 命令和配置文件去操作 kubernetes 资源

  ```bash
  kubectl apply -f nginx-pod.yaml
  ```

| 类型           | 操作对象 | 适用环境 | 优点                                       | 缺点                             |
| -------------- | -------- | -------- | ------------------------------------------ | -------------------------------- |
| 命令式对象管理 | 对象     | 测试     | 简单，常用于查看                           | 只能操作活动对象，无法审计、跟踪 |
| 命令式对象配置 | 文件     | 开发     | 可以审计、跟踪                             | 项目大时，配置文件多，操作麻烦   |
| 声明式对象配置 | 目录     | 开发     | 支持目录操作，可以执行目录下所有 yaml 文件 | 意外情况下难以调试               |

#### 1.命令式对象管理

**kubectl命令**

`kubectl`是 kubernetes 集群的命令行工具，通过它能够对集群本身进行管理，并能够在集群上进行容器化应用的安装部署。`kubectl`命令的语法如下：

```bash
kubectl [command] [type] [name] [flags]
```

- `comand`：指定要对资源执行的操作，例如 create、get、delete

- `type`：指定资源类型，比如 deployment、pod、service

- `name`：指定资源的名称，名称大小写敏感

- `flags`：指定额外的可选参数

```bash
# 查看所有pod
kubectl get pod 

# 查看某个pod
kubectl get pod pod_name

# 查看某个pod,以yaml格式展示结果
kubectl get pod pod_name -o yaml
```

**资源类型**

kubernetes 中所有的内容都抽象为资源，可以通过下面的命令进行查看：

```bash
kubectl api-resources
```

经常使用的资源有下面这些：

| 资源分类      | 资源名称                 | 缩写    | 资源作用        |
| ------------- | ------------------------ | ------- | --------------- |
| 集群级别资源  | nodes                    | no      | 集群组成部分    |
| namespaces    | ns                       | 隔离Pod |                 |
| pod资源       | pods                     | po      | 装载容器        |
| pod资源控制器 | replicationcontrollers   | rc      | 控制pod资源     |
|               | replicasets              | rs      | 控制pod资源     |
|               | deployments              | deploy  | 控制pod资源     |
|               | daemonsets               | ds      | 控制pod资源     |
|               | jobs                     |         | 控制pod资源     |
|               | cronjobs                 | cj      | 控制pod资源     |
|               | horizontalpodautoscalers | hpa     | 控制pod资源     |
|               | statefulsets             | sts     | 控制pod资源     |
| 服务发现资源  | services                 | svc     | 统一pod对外接口 |
|               | ingress                  | ing     | 统一pod对外接口 |
| 存储资源      | volumeattachments        |         | 存储            |
|               | persistentvolumes        | pv      | 存储            |
|               | persistentvolumeclaims   | pvc     | 存储            |
| 配置资源      | configmaps               | cm      | 配置            |
|               | secrets                  |         | 配置            |

**操作**

kubernetes 允许对资源进行多种操作，可以通过`--help`查看详细的操作命令

```bash
kubectl --help
```

经常使用的操作有下面这些：

| 命令分类   | 命令         | 翻译                        | 命令作用                     |
| ---------- | ------------ | --------------------------- | ---------------------------- |
| 基本命令   | create       | 创建                        | 创建一个资源                 |
|            | edit         | 编辑                        | 编辑一个资源                 |
|            | get          | 获取                        | 获取一个资源                 |
|            | patch        | 更新                        | 更新一个资源                 |
|            | delete       | 删除                        | 删除一个资源                 |
|            | explain      | 解释                        | 展示资源文档                 |
| 运行和调试 | run          | 运行                        | 在集群中运行一个指定的镜像   |
|            | expose       | 暴露                        | 暴露资源为Service            |
|            | describe     | 描述                        | 显示资源内部信息             |
|            | logs         | 日志输出容器在 pod 中的日志 | 输出容器在 pod 中的日志      |
|            | attach       | 缠绕进入运行中的容器        | 进入运行中的容器             |
|            | exec         | 执行容器中的一个命令        | 执行容器中的一个命令         |
|            | cp           | 复制                        | 在Pod内外复制文件            |
|            | rollout      | 首次展示                    | 管理资源的发布               |
|            | scale        | 规模                        | 扩(缩)容Pod的数量            |
|            | autoscale    | 自动调整                    | 自动调整Pod的数量            |
| 高级命令   | apply        | rc                          | 通过文件对资源进行配置       |
|            | label        | 标签                        | 更新资源上的标签             |
| 其他命令   | cluster-info | 集群信息                    | 显示集群信息                 |
|            | version      | 版本                        | 显示当前Server和Client的版本 |

下面以一个 namespace / pod 的创建和删除简单演示下命令的使用：

```bash
# 创建一个namespace
[root@master ~]# kubectl create namespace dev
namespace/dev created

# 获取namespace
[root@master ~]# kubectl get ns
NAME              STATUS   AGE
default           Active   21h
dev               Active   21s
kube-node-lease   Active   21h
kube-public       Active   21h
kube-system       Active   21h

# 在此 namespace 下创建并运行一个 nginx 的 Pod
[root@master ~]# kubectl run pod --image=nginx:latest -n dev
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/pod created

# 查看新创建的 pod
[root@master ~]# kubectl get pod -n dev
NAME  READY   STATUS    RESTARTS   AGE
pod   1/1     Running   0          21s

# 删除指定的 pod
[root@master ~]# kubectl delete pod pod -n dev
pod "pod" deleted

# 删除指定的 namespace
[root@master ~]# kubectl delete ns dev
namespace "dev" deleted
```

#### 2.命令式对象配置

命令式对象配置就是使用命令配合配置文件一起来操作 kubernetes 资源。

1） 创建一个 nginxpod.yaml，内容如下：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev

---

apiVersion: v1
kind: Pod
metadata:
  name: nginxpod
  namespace: dev
spec:
  containers:
  - name: nginx-containers
    image: nginx:latest
```

2）执行`create`命令，创建资源：

```bash
[root@master ~]# kubectl create -f nginxpod.yaml
namespace/dev created
pod/nginxpod created
```

此时发现创建了两个资源对象，分别是 namespace 和 pod。

3）执行`get`命令，查看资源：

```bash
[root@master ~]#  kubectl get -f nginxpod.yaml
NAME            STATUS   AGE
namespace/dev   Active   18s

NAME            READY   STATUS    RESTARTS   AGE
pod/nginxpod    1/1     Running   0          17s
```

这样就显示了两个资源对象的信息。

4）执行`delete`命令，删除资源：

```bash
[root@master ~]# kubectl delete -f nginxpod.yaml
namespace "dev" deleted
pod "nginxpod" deleted
```

此时发现两个资源对象被删除了。

**总结**

命令式对象配置的方式操作资源，可以简单的认为：命令  +  yaml 配置文件（里面是命令需要的各种参数）。

#### 3.声明式对象配置

声明式对象配置跟命令式对象配置很相似，但是它只有一个命令`apply`。

```bash
# 首先执行一次 kubectl apply -f yaml 文件，发现创建了资源
[root@master ~]#  kubectl apply -f nginxpod.yaml
namespace/dev created
pod/nginxpod created

# 再次执行一次，发现说资源没有变动
[root@master ~]#  kubectl apply -f nginxpod.yaml
namespace/dev unchanged
pod/nginxpod unchanged
```

**总结**

其实声明式对象配置就是使用 apply 描述一个资源最终的状态（在 yaml 中定义状态），使用 apply 操作资源：

- 如果资源不存在，就创建，相当于`kubectl create`
- 如果资源已存在，就更新，相当于`kubectl patch`（使用`kubectl create`则会报错）

#### 4.总结

- 创建/更新资源：使用声明式对象配置`kubectl apply -f XXX.yaml`

- 删除资源：使用命令式对象配置`kubectl delete -f XXX.yaml`

- 查询资源：使用命令式对象管理`kubectl get(describe) 资源名称`

**在 node 节点上运行 kubectl**

一般情况下，在 node 节点执行任意`kubectl`命令，例如：`kubectl get nodes`，命令会报错。

`kubectl`的运行是需要进行配置的，需要将`admin.conf`文件上传到 node 节点的`/etc/kubernetes`目录下，在 master 节点上执行下面操作：

```bash
scp -r /etc/kubernetes/admin.conf node1:/etc/kubernetes/admin.conf
```

再在 node 节点配置环境变量

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

## 4.资源的操作

查看所有的资源：`kubectl get all -A`

### 4.1 Namespace

namespace 是 kubernetes 系统中的一种非常重要资源，它的主要作用是用来实现**多套环境的资源隔离**或者**多租户的资源隔离**。

默认情况下，kubernetes 集群中的所有的 pod 都是可以相互访问的。可将两个 pod 划分到不同的 namespace 下可以禁止访问。kubernetes 通过将集群内部的资源分配到不同的 namespace 中，可以形成逻辑上的"组"，以方便不同的组的资源进行隔离使用和管理。

可以通过 kubernetes 的授权机制，将不同的 namespace 交给不同租户进行管理，这样就实现了多租户的资源隔离。此时还能结合 kubernetes 的资源配额机制，限定不同租户能占用的资源，例如 CPU 使用量、内存使用量等等，来实现租户可用资源的管理。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200407100850484-5c0ca682512b5133cb5f1fc83161e52e-83104f.png" alt="image-20200407100850484" style="zoom:50%;" />

kubernetes 在集群启动之后，会默认创建几个 namespace

```bash
[root@master ~]# kubectl  get namespace
NAME              STATUS   AGE
default           Active   45h     #  所有未指定 namespace 的对象都会被分配在 default 命名空间
kube-node-lease   Active   45h     #  集群节点之间的心跳维护，v1.13 开始引入
kube-public       Active   45h     #  此命名空间下的资源可以被所有人访问（包括未认证用户）
kube-system       Active   45h     #  所有由 kubernetes 系统创建的资源都处于这个命名空间
```

#### 1.查看

```bash
# 1 查看所有的 ns
[root@master ~]# kubectl get ns
NAME              STATUS   AGE
default           Active   45h
kube-node-lease   Active   45h
kube-public       Active   45h     
kube-system       Active   45h     

# 2 查看指定的 ns
[root@master ~]# kubectl get ns default
NAME      STATUS   AGE
default   Active   45h

# 3 指定输出格式，如 wide、json、yaml
[root@master ~]# kubectl get ns default -o yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2021-05-08T04:44:16Z"
  name: default
  resourceVersion: "151"
  selfLink: /api/v1/namespaces/default
  uid: 7405f73a-e486-43d4-9db6-145f1409f090
spec:
  finalizers:
  - kubernetes
status:
  phase: Active
  
# 4 查看 ns 详情
[root@master ~]# kubectl describe ns default
Name:         default
Labels:       <none>
Annotations:  <none>
Status:       Active  # Active：命名空间正在使用中；Terminating：正在删除命名空间

# ResourceQuota 针对 namespace 做的资源限制
# LimitRange 针对 namespace 中的每个组件做的资源限制
No resource quota.
No LimitRange resource.
```

#### 2.创建

```bash
# 创建 namespace
[root@master ~]# kubectl create ns dev
namespace/dev created
```

#### 3.删除

```bash
# 删除 namespace
[root@master ~]# kubectl delete ns dev
namespace "dev" deleted
```

#### 4.配置方式

首先准备一个 yaml 文件：ns-dev.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

然后就可以执行对应的创建和删除命令了：

创建：`kubectl create -f ns-dev.yaml`

删除：`kubectl delete -f ns-dev.yaml`

### 4.2 Pod

pod 是 kubernetes 集群进行管理的最小单元，程序要运行必须部署在容器中，而容器必须存在于 pod 中。

pod 可以认为是容器的封装，一个 pod 中可以存在一个或者多个容器。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200407121501907.png" alt="image-20200407121501907" style="zoom:67%;" />

kubernetes 在集群启动之后，集群中的各个组件也都是以 pod 方式运行的。可以通过下面命令查看：

```bash
[root@master kubernetes]# kubectl get pod -n kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-787d4945fb-5v6l4         1/1     Running   0          135m
coredns-787d4945fb-k7pm5         1/1     Running   0          135m
etcd-master                      1/1     Running   6          135m
kube-apiserver-master            1/1     Running   6          135m
kube-controller-manager-master   1/1     Running   6          135m
kube-proxy-7l9lc                 1/1     Running   0          135m
kube-proxy-jh9hj                 1/1     Running   0          135m
kube-proxy-nb8wl                 1/1     Running   0          135m
kube-scheduler-master            1/1     Running   6          135m

```

#### 1.创建并运行

需要提前创建好 dev 命名空间。

```bash
# 命令格式： kubectl run (pod名称) [参数] 
# --image  指定Pod的镜像
# --port   指定端口
# --namespace  指定namespace
[root@master ~]# kubectl run nginx --image=nginx:latest --port=80 --namespace dev
pod/nginx created
```

> **注意**
>
> 新版由于`run`不会创建控制器，所以可以直接创建一个名称为`nginx `的 pod，而旧版中则会创建一个名称为`nginx`的 pod 控制器，再由控制器创建一个`nginx+随机数`命名的 pod。

#### 2.查看pod信息

```bash
[root@master ~]# kubectl get pod -n dev
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          96s
[root@master ~]# kubectl describe pod nginx -n dev
...
```

#### 3.访问Pod

```bash
# 获取 pod IP
[root@master ~]# kubectl get pod -n dev -o wide
NAME    READY   STATUS    RESTARTS   AGE     IP           NODE    NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          2m17s   10.244.1.5   node2   <none>           <none>
# 通过 pod IP 和指定的端口访问
[root@master ~]# curl 10.244.1.5:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

#### 4.删除指定pod

```bash
# 删除
[root@master ~]# kubectl delete pod nginx -n dev
pod "nginx" deleted

[root@master ~]# kubectl get pod -n dev
No resources found in dev namespace.
```

> **注意**
>
> 新版可以使用`delete pod podName`直接删除，因为使用`run`命令不会创建 pod 控制器，使用 deploy 才会创建控制器；
>
> 而旧版由于创建的是 pod 控制器，所以使出 pod 的话控制器会重建一个 pod，直接删除控制器即可：`kubectl delete deploy nginx -n dev`

#### 5.配置操作

创建一个pod-nginx.yaml，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: dev
spec:
  containers:
  - image: nginx:latest
    name: pod
    ports:
    - name: nginx-port
      containerPort: 80
      protocol: TCP
```

然后就可以执行对应的创建和删除命令了：

创建：`kubectl create -f pod-nginx.yaml`

删除：`kubectl delete -f pod-nginx.yaml`

### 4.3 Label

Label 是 kubernetes 系统中的一个重要概念。它的作用就是在资源上添加标识，用来对它们进行区分和选择。

Label 的特点：

- 一个 Label 会以 key/value 键值对的形式附加到各种对象上，如 Node、Pod、Service 等等
- 一个资源对象可以定义任意数量的 Label ，同一个 Label 也可以被添加到任意数量的资源对象上去
- Label 通常在资源对象定义时确定，当然也可以在对象创建后动态添加或者删除

可以通过 Label 实现资源的多维度分组，以便灵活、方便地进行资源分配、调度、配置、部署等管理工作。

> 一些常用的 Label 示例如下：
>
> - 版本标签："version":"release", "version":"stable"......
> - 环境标签："environment":"dev"，"environment":"test"，"environment":"pro"
> - 架构标签："tier":"frontend"，"tier":"backend"

标签定义完毕之后，还要考虑到标签的选择，这就要使用到 Label Selector，即：

- Label 用于给某个资源对象定义标识

- Label Selector 用于查询和筛选拥有某些标签的资源对象

当前有两种 Label Selector：

- 基于等式的 Label Selector

  `name = slave`：选择所有包含Label中key="name"且value="slave"的对象

  `env != production`：选择所有包括 Label 中的 key="env" 且 value 不等于 "production" 的对象

- 基于集合的 Label Selector

  `name in (master, slave)`：选择所有包含 Label 中的 key="name" 且 value="master" 或 "slave" 的对象

  `name not in (frontend)`：选择所有包含 Label 中的 key="name" 且 value 不等于 "frontend" 的对象

标签的选择条件可以使用多个，此时将多个 Label Selector 进行组合，使用逗号`,`进行分隔即可。例如：

- `name=slave, env!=production`
- `name not in (frontend), env!=production`

##### 1.命令方式

```bash
# 为已有 pod 资源打标签
[root@master ~]# kubectl label pod nginx-pod version=1.0 -n dev
pod/nginx-pod Labeled

# 为 pod 资源更新标签
[root@master ~]# kubectl label pod nginx-pod version=2.0 -n dev --overwrite
pod/nginx-pod Labeled

# 查看标签
[root@master ~]# kubectl get pod nginx-pod -n dev --show-labels
NAME        READY   STATUS    RESTARTS   AGE   LABELS
nginx-pod   1/1     Running   0          10m   version=2.0

# 筛选标签
[root@master ~]# kubectl get pod -n dev -l version=2.0  --show-labels
NAME        READY   STATUS    RESTARTS   AGE   LABELS
nginx-pod   1/1     Running   0          17m   version=2.0
[root@master ~]# kubectl get pod -n dev -l version!=2.0 --show-labels
No resources found in dev namespace.

#删除标签
[root@master ~]# kubectl label pod nginx-pod version- -n dev
pod/nginx-pod Labeled
```

##### 2.配置方式 - metadata.labels

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: dev
  labels:
    version: "3.0" 
    env: "test"
spec:
  containers:
  - image: nginx:latest
    name: pod
    ports:
    - name: nginx-port
      containerPort: 80
      protocol: TCP
```

然后就可以执行对应的更新命令了：`kubectl apply -f pod-nginx.yaml`（如果存在 nginx 的 pod 则会更新标签）

### 4.4 Deployment

在 kubernetes 中，Pod 是最小的控制单元，但是 kubernetes 很少直接控制 Pod，一般都是通过 Pod 控制器来完成的。Pod 控制器用于 pod 的管理，确保 pod 资源符合预期的状态，当 pod 的资源出现故障时，会尝试进行重启或重建 pod。

在 kubernetes 中 Pod 控制器的种类有很多，本章节只介绍一种：Deployment。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200408193950807-cbf7a130388403bebb7287c15f57a377-59df38.png" alt="image-20200408193950807" style="zoom:67%;" />

#### 1.创建并运行

```bash
# 命令格式: kubectl create deployment 名称  [参数] 
# --image  指定 pod 的镜像
# --port   指定端口
# --replicas  指定创建 pod 数量
# --namespace  指定 namespace

# 旧版本：kubectl run nginx --image=nginx:latest --port=80 --replicas=3 -n dev
[root@master ~]# kubectl create deployment my-dep --image=nginx:latest --port=80 --replicas=3 -n dev
deployment.apps/my-dep created

# 查看创建的 Pod
[root@master ~]# kubectl get pods -n dev
NAME                      READY   STATUS    RESTARTS   AGE
my-dep-67c68dbfdf-2zscj   1/1     Running   0          59s
my-dep-67c68dbfdf-69rjl   1/1     Running   0          8m12s
my-dep-67c68dbfdf-z4v97   1/1     Running   0          8m12s

# 查看deployment 的信息
[root@master ~]# kubectl get deployment -n dev
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
my-dep   3/3     3            3           7m52s

# UP-TO-DATE：成功升级的副本数量
# AVAILABLE：可用副本的数量
# SELECTOR: 痛着标签选择器来分组各个 pod

[root@master ~]# kubectl get deploy -n dev -o wide
NAME     READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
my-dep   3/3     3            3           11m   nginx        nginx:latest   app=my-dep

# 查看 deployment 的详细信息
[root@master ~]# kubectl describe deploy my-dep -n dev
Name:                   my-dep
Namespace:              dev
CreationTimestamp:      Sun, 29 Jan 2023 05:42:02 -0800
Labels:                 app=my-dep
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=my-dep
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=my-dep
  Containers:
   nginx:
    Image:        nginx:latest
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   my-dep-67c68dbfdf (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  12m   deployment-controller  Scaled up replica set my-dep-67c68dbfdf to 3
  
# 删除 
[root@master ~]# kubectl delete deploy my-dep -n dev
deployment.apps "nginx" deleted
```

#### 2.配置操作

创建一个 deploy-nginx.yaml，内容如下：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-dep
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-dep
  template:
    metadata:
      labels:
        app: my-dep
    spec:
      containers:
      - image: nginx:latest
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
```

然后就可以执行对应的创建和删除命令了：

创建：`kubectl create -f deploy-nginx.yaml`

删除：`kubectl delete -f deploy-nginx.yaml`

### 4.5 Service

通过上节课的学习，已经能够利用 Deployment 来创建一组 Pod 来提供具有高可用性的服务。

虽然每个 Pod 都会分配一个单独的 Pod IP，然而却存在如下两问题：

- Pod IP 会随着 Pod 的重建产生变化
- Pod IP 仅仅是集群内可见的虚拟 IP，外部无法访问

这样对于访问这个服务带来了难度。因此，kubernetes 设计了 Service 来解决这个问题。

Service 可以看作是一组同类 Pod **对外的访问接口**。借助 Service，应用可以方便地实现服务发现和负载均衡。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200408194716912-934ff8293c302c586158d2f8fd50bd3c-1f507e.png" alt="image-20200408194716912" style="zoom: 67%;" />

#### 1.集群内部访问

```bash
# 暴露 Service
# 使用 my-dep 的标签来指定可访问的 pod
[root@master ~]# kubectl expose deploy my-dep --name=svc-nginx --type=ClusterIP --port=80 --target-port=80 -n dev
service/svc-nginx exposed

# 查看 Service
[root@master ~]# kubectl get svc svc-nginx -n dev -o wide
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
svc-nginx   ClusterIP   10.105.165.86   <none>        80/TCP    6s    app=my-dep

# 这里产生了一个 CLUSTER-IP，这就是 service 的 IP，在 Service 的生命周期中，这个地址是不会变动的
# 可以通过这个 IP 访问当前 service 对应的 POD
[root@master ~]# curl 10.105.165.86
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
</head>
<body>
<h1>Welcome to nginx!</h1>
.......
</body>
</html>
```

#### 2.集群外部访问

```bash
# 上面创建的 Service 的 type 类型为 ClusterIP，这个ip地址只用集群内部可访问
# 如果需要创建外部也可以访问的 Service，需要修改 type 为 NodePort
[root@master ~]# kubectl expose deploy my-dep --name=svc-nginx2 --type=NodePort --port=80 --target-port=80 -n dev
service/svc-nginx2 exposed

# 此时查看，会发现出现了 NodePort 类型的 Service，而且有一对 Port（80:31928/TC）
[root@master ~]# kubectl get svc  svc-nginx2  -n dev -o wide
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE    SELECTOR
svc-nginx2   NodePort   10.106.213.24   <none>        80:31124/TCP   2m1s   app=my-dep

# 接下来就可以通过集群外的主机访问节点 IP:31124 访问服务了
# 例如在的电脑主机上通过浏览器访问下面的地址
http://192.168.11.101:31124/
```

#### 3.删除Service

```bash
[root@master ~]# kubectl delete svc svc-nginx2 -n dev 
service "svc-nginx2" deleted
```

#### 4.配置方式

创建一个svc-nginx.yaml，内容如下：

```bash
apiVersion: v1
kind: Service
metadata:
  name: svc-nginx
  namespace: dev
spec:
  clusterIP: 10.109.179.231 #固定 svc 的内网 ip
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: my-dep
  type: ClusterIP
```

然后就可以执行对应的创建和删除命令了：

创建：`kubectl create -f svc-nginx.yaml`

删除：`kubectl delete -f svc-nginx.yaml`