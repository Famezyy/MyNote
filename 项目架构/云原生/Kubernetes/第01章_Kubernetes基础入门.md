# 第01章_Kubernetes基础入门

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

官方地址：[https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/](https://gitee.com/link?target=https%3A%2F%2Fkubernetes.io%2Fdocs%2Freference%2Fsetup-tools%2Fkubeadm%2Fkubeadm%2F)

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

> **问题**
>
> 出现以下错误时`[ERROR CRI]: container runtime is not running`的解决方案：
>
> ```bash
> rm -rf /etc/containerd/config.toml
> systemctl restart containerd
> ```

等初始化完成后会显示 node 节点加入集群的命令，只需要在 node 节点上执行即可。

> **在 master 上查看节点信息**
>
> ```bash
> kubectl get nodes
> ```
>
> **需要重置时**
>
> ```bash
> kubeadm reset
> rm -rf /etc/cni/net.d
> ipvsadm --clear
> rm -rf $HOME/.kube/config
> ip link delete flannel.1
> ip link delete cni0
> systemctl restart containerd
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

在 kubernetes 中，所有的内容都抽象为**资源**，用户需要通过操作资源来管理 kubernetes。

kubernetes 的本质上就是一个集群系统，用户可以在集群中部署各种服务，所谓的部署服务，其实就是在 kubernetes 集群中运行一个个的容器，并将指定的程序跑在容器中。

kubernetes 的最小管理单元是`pod`而不是容器，所以只能将容器放在`pod`中，而 kubernetes 一般也不会直接管理`pod`，而是通过`pod控制器`来管理`pod`的。

`pod`可以提供服务之后，就要考虑如何访问`pod`中服务，kubernetes 提供了`service`资源实现这个功能。

当然，如果`pod`中程序的数据需要持久化，kubernetes 还提供了各种`存储`系统。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200406225334627-102ffc00a84026635682f8cfd33e6d1e-ef403f.png" alt="img" style="zoom: 67%;" />

学习 kubernetes 的核心，就是学习如何对集群上的`Pod、Pod控制器、Service、存储`等各种资源进行操作。