# 第04章_Service

## 1.Service介绍

在 kubernetes 中，pod 是应用程序的载体，我们可以通过 pod 的 ip 来访问应用程序，但是 pod 的 ip 地址不是固定的，这也就意味着不方便直接采用 pod 的 ip 对服务进行访问。

为了解决这个问题，kubernetes 提供了 Service 资源，Service 会对提供同一个服务的多个 pod 进行聚合，并且提供一个统一的入口地址。通过访问 Service 的入口地址就能访问到后面的 pod 服务。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200408194716912-1626783758946.png" alt="img" style="zoom: 67%;" />

Service 在很多情况下只是一个概念，真正起作用的其实是 kube-proxy 服务进程，每个 Node 节点上都运行着一个 kube-proxy 服务进程。当创建 Service 的时候会通过 api-server 向 etcd 写入创建的 Service 的信息，而 kube-proxy 会基于监听的机制发现这种 Service 的变动，然后**它会将最新的 Service 信息转换成对应的访问规则**。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200509121254425.png" alt="img" style="zoom: 67%;" />

```bash
[root@node1 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.97.97.97:80 rr
  -> 10.244.1.39:80               Masq    1      0          0
  -> 10.244.1.40:80               Masq    1      0          0
  -> 10.244.2.33:80               Masq    1      0          0
```

10.97.97.97:80 是 service 提供的访问入口，当访问这个入口的时候，可以发现后面有三个 pod 的服务在等待调用，kube-proxy 会基于 rr（轮询）的策略，将请求分发到其中一个 pod 上去。这个规则会同时在集群内的所有节点上都生成，所以在任何一个节点，访问都可以。

### 1.1 userspace模式

userspace 模式下，kube-proxy 会为每一个 Service 创建一个监听端口，发向 Cluster IP 的请求被 Iptables 规则重定向到 kube-proxy 监听的端口上，kube-proxy 根据 LB 算法选择一个提供服务的 Pod 并和其建立链接，以将请求转发到 Pod 上。该模式下，kube-proxy 充当了一个四层负责均衡器的角色。由于 kube-proxy 运行在 userspace 中，在进行转发处理时会增加内核和用户空间之间的数据拷贝，虽然比较稳定，但是效率比较低。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200509151424280.png" alt="img" style="zoom:50%;" />

### 1.2 iptables模式

iptables 模式下，kube-proxy 为 service 后端的每个 Pod 创建对应的 iptables 规则，直接将发向 Cluster IP 的请求重定向到一个 Pod IP。该模式下 kube-proxy 不承担四层负责均衡器的角色，只负责创建 iptables 规则。该模式的优点是较 userspace 模式效率更高，但不能提供灵活的 LB 策略，当后端 Pod 不可用时也无法进行重试。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200509152947714.png" alt="img" style="zoom:50%;" />

### 1.3 ipvs模式

ipvs 模式和 iptables 类似，kube-proxy 监控 Pod 的变化并创建相应的 ipvs 规则。ipvs 相对 iptables 转发效率更高。除此以外，ipvs 支持更多的 LB 算法。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200509153731363.png" alt="img" style="zoom:50%;" />

此模式必须安装 ipvs 内核模块，否则会降级为 iptables（见<a href="第01章_Kubernetes基础.md">第一章节</a>）。

```bash
# 开启 ipvs
[root@k8s-master01 ~]# kubectl edit cm kube-proxy -n kube-system
# 修改 mode: "ipvs"
[root@k8s-master01 ~]# kubectl delete pod -l k8s-app=kube-proxy -n kube-system
[root@node1 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.97.97.97:80 rr
  -> 10.244.1.39:80               Masq    1      0          0
  -> 10.244.1.40:80               Masq    1      0          0
  -> 10.244.2.33:80               Masq    1      0          0
```

## 2.Service类型

Service 的资源清单文件：

```yaml
kind: Service  # 资源类型
apiVersion: v1  # 资源版本
metadata: # 元数据
  name: service # 资源名称
  namespace: dev # 命名空间
spec: # 描述
  selector: # 标签选择器，用于确定当前 service 代理哪些 pod
    app: nginx
  type: # Service类型，指定 service 的访问方式
  clusterIP:  # 虚拟服务的 ip 地址
  sessionAffinity: # session 亲和性，支持 ClientIP、None 两个选项
  ports: # 端口信息
    - protocol: TCP 
      port: 3017  # service 端口
      targetPort: 5003 # pod 端口
      nodePort: 31122 # 主机端口
```

`type`类型：

- `ClusterIP`：默认值，它是 Kubernetes 系统自动分配的虚拟 IP，只能在集群内部访问
- `NodePort`：将 Service 通过指定的 Node 上的端口暴露给外部，通过此方法，就可以在集群外部访问服务
- `LoadBalancer`：使用外接负载均衡器完成到服务的负载分发，注意此模式需要外部云环境支持
- `ExternalName`：把集群外部的服务引入集群内部，直接使用

### 2.1 实验环境准备

在使用 service 之前，首先利用 Deployment 创建出 3 个 pod，注意要为 pod 设置`app=nginx-pod`的标签

创建 deployment.yaml，内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80	
```

```bash
[root@k8s-master01 ~]# kubectl create -f deployment.yaml
deployment.apps/pc-deployment created

# 查看 pod 详情
[root@k8s-master01 ~]# kubectl get pods -n dev -o wide --show-labels
NAME                             READY   STATUS     IP            NODE     LABELS
pc-deployment-66cb59b984-8p84h   1/1     Running    10.244.1.39   node1    app=nginx-pod
pc-deployment-66cb59b984-vx8vx   1/1     Running    10.244.2.33   node2    app=nginx-pod
pc-deployment-66cb59b984-wnncx   1/1     Running    10.244.1.40   node1    app=nginx-pod

# 为了方便后面的测试，修改下三台 nginx 的 index.html 页面（三台修改的 IP 地址不一致）
# kubectl exec -it pc-deployment-66cb59b984-8p84h -n dev /bin/sh
# echo "10.244.1.39" > /usr/share/nginx/html/index.html

# 修改完毕之后，访问测试
[root@k8s-master01 ~]# curl 10.244.1.39
10.244.1.39
[root@k8s-master01 ~]# curl 10.244.2.33
10.244.2.33
[root@k8s-master01 ~]# curl 10.244.1.40
10.244.1.40
```

### 2.2 ClusterIP类型

创建 service-clusterip.yaml 文件

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-clusterip
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: 10.97.97.97 # service 的 ip 地址，如果不写，默认会生成一个
  type: ClusterIP
  ports:
  - port: 80  # Service 端口，可随便设置，这里为了统一也设置成 80
    targetPort: 80 # pod 端口，要与 pod 中运行的服务端口相同
```

```bash
# 创建 service
[root@k8s-master01 ~]# kubectl create -f service-clusterip.yaml
service/service-clusterip created

# 查看 service
[root@k8s-master01 ~]# kubectl get svc -n dev -o wide
NAME                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service-clusterip   ClusterIP   10.97.97.97   <none>        80/TCP    13s   app=nginx-pod

# 查看 service 的详细信息
# 在这里有一个 Endpoints 列表，里面就是当前 service 可以负载到的服务入口
[root@k8s-master01 ~]# kubectl describe svc service-clusterip -n dev
Name:              service-clusterip
Namespace:         dev
Labels:            <none>
Annotations:       <none>
Selector:          app=nginx-pod
Type:              ClusterIP
IP:                10.97.97.97
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.1.39:80,10.244.1.40:80,10.244.2.33:80
Session Affinity:  None
Events:            <none>

# 查看 ipvs 的映射规则
[root@k8s-master01 ~]# ipvsadm -Ln
TCP  10.97.97.97:80 rr
  -> 10.244.1.39:80               Masq    1      0          0
  -> 10.244.1.40:80               Masq    1      0          0
  -> 10.244.2.33:80               Masq    1      0          0

# 访问 10.97.97.97:80 观察效果
[root@k8s-master01 ~]# curl 10.97.97.97:80
10.244.2.33
```

> **Endpoint**
>
> Endpoint 也是 kubernetes 中的一个资源对象，存储在 etcd 中，用来记录一个 service 对应的所有 pod 的访问地址，它是根据 service 配置文件中 selector 描述产生的。
>
> 一个 Service 由一组 Pod 组成，这些 Pod 通过 Endpoints 暴露出来，**Endpoints 是实现实际服务的端点集合**。换句话说，service 和 pod 之间的联系是通过 endpoints 实现的。
>
> <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200509191917069.png" alt="image-20200509191917069" style="zoom: 67%;" />

#### 负载分发策略

对 Service 的访问被分发到了后端的 Pod 上去，目前 kubernetes 提供了两种负载分发策略：

- 如果不定义，默认使用 kube-proxy 的策略，比如随机、轮询
- 基于客户端地址的会话保持模式，即来自同一个客户端发起的所有请求都会转发到固定的一个 Pod 上，使用此模式可以在 spec 中添加`sessionAffinity:ClientIP`选项

```bash
# 查看 ipvs 的映射规则【rr 轮询】
[root@k8s-master01 ~]# ipvsadm -Ln
TCP  10.97.97.97:80 rr
  -> 10.244.1.39:80               Masq    1      0          0
  -> 10.244.1.40:80               Masq    1      0          0
  -> 10.244.2.33:80               Masq    1      0          0

# 循环访问测试
[root@k8s-master01 ~]# while true;do curl 10.97.97.97:80; sleep 5; done;
10.244.1.40
10.244.1.39
10.244.2.33
10.244.1.40
10.244.1.39
10.244.2.33

# 修改分发策略----sessionAffinity: ClientIP

# 查看ipvs规则【persistent 代表持久】
[root@k8s-master01 ~]# ipvsadm -Ln
TCP  10.97.97.97:80 rr persistent 10800
  -> 10.244.1.39:80               Masq    1      0          0
  -> 10.244.1.40:80               Masq    1      0          0
  -> 10.244.2.33:80               Masq    1      0          0

# 循环访问测试
[root@k8s-master01 ~]# while true;do curl 10.97.97.97; sleep 5; done;
10.244.2.33
10.244.2.33
10.244.2.33
  
# 删除 service
[root@k8s-master01 ~]# kubectl delete -f service-clusterip.yaml
service "service-clusterip" deleted
```

### 2.3 HeadLiness类型

在某些场景中，开发人员可能不想使用 Service 提供的负载均衡功能，而希望自己来控制负载均衡策略，针对这种情况，kubernetes 提供了 HeadLiness Service，这类 Service 不会分配 Cluster IP，如果想要访问 Service，只能通过 Service 的域名进行查询。

创建 service-headliness.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-headliness
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: None # 将 clusterIP 设置为 None，即可创建 headliness Service
  type: ClusterIP
  ports:
  - port: 80    
    targetPort: 80
```

```bash
# 创建 service
[root@k8s-master01 ~]# kubectl create -f service-headliness.yaml
service/service-headliness created

# 获取 service， 发现 CLUSTER-IP 未分配
[root@k8s-master01 ~]# kubectl get svc service-headliness -n dev -o wide
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service-headliness   ClusterIP   None         <none>        80/TCP    11s   app=nginx-pod

# 查看 Service 详情
[root@k8s-master01 ~]# kubectl describe svc service-headliness  -n dev
Name:              service-headliness
Namespace:         dev
Labels:            <none>
Annotations:       <none>
Selector:          app=nginx-pod
Type:              ClusterIP
IP:                None
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.1.39:80,10.244.1.40:80,10.244.2.33:80
Session Affinity:  None
Events:            <none>

# 查看域名的解析情况
[root@k8s-master01 ~]# kubectl exec -it pc-deployment-66cb59b984-8p84h -n dev /bin/bash
/ # cat /etc/resolv.conf
nameserver 10.96.0.10
search dev.svc.cluster.local svc.cluster.local cluster.local

[root@k8s-master01 ~]# dig @10.96.0.10 service-headliness.dev.svc.cluster.local
service-headliness.dev.svc.cluster.local. 30 IN A 10.244.1.40
service-headliness.dev.svc.cluster.local. 30 IN A 10.244.1.39
service-headliness.dev.svc.cluster.local. 30 IN A 10.244.2.33
```

可以看到，HeadLiness 类型下的 IP 分配到了`10.244.1.40`、`10.244.1.39`、`10.244.2.33`

### 2.4 NodePort类型

在之前的样例中，创建的 Service 的 ip 地址只有集群内部才可以访问，如果希望将 Service 暴露给集群外部使用，那么就要使用到另外一种类型的 Service，称为 NodePort 类型。NodePort 的工作原理其实就是**将 service 的端口映射到 Node 的一个端口上**，然后就可以通过`NodeIp:NodePort`来访问 service 了。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200620175731338.png" alt="img" style="zoom: 80%;" />

创建 service-nodeport.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-nodeport
  namespace: dev
spec:
  selector:
    app: nginx-pod
  type: NodePort # service　类型
  ports:
  - port: 80
    nodePort: 30002 # 指定绑定的 node 的端口（默认的取值范围是：30000-32767）, 如果不指定，会默认分配
    targetPort: 80
```

```bash
# 创建 service
[root@k8s-master01 ~]# kubectl create -f service-nodeport.yaml
service/service-nodeport created

# 查看 service
[root@k8s-master01 ~]# kubectl get svc -n dev -o wide
NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)       SELECTOR
service-nodeport   NodePort   10.105.64.191   <none>        80:30002/TCP  app=nginx-pod

# 接下来可以通过电脑主机的浏览器去访问集群中任意一个 nodeip 的 30002 端口，即可访问到 pod
```

### 2.5 LoadBalancer类型

LoadBalancer 和 NodePort 很相似，目的都是向外部暴露一个端口，区别在于 LoadBalancer 会在集群的外部再来做一个负载均衡设备，而这个设备**需要外部环境支持**的，外部服务发送到这个设备上的请求，会被设备负载之后转发到集群中。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200510103945494.png" alt="img" style="zoom:67%;" />

### 2.6 ExternalName类型

ExternalName 类型的 Service 用于引入集群外部的服务，它通过`externalName`属性指定外部一个服务的地址，然后在集群内部访问此 service 就可以访问到外部的服务了。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200510113311209.png" alt="img" style="zoom:80%;" />

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-externalname
  namespace: dev
spec:
  type: ExternalName # service 类型
  externalName: www.baidu.com  #改成 ip 地址也可以
```

```bash
# 创建 service
[root@k8s-master01 ~]# kubectl  create -f service-externalname.yaml
service/service-externalname created

# 域名解析
[root@k8s-master01 ~]# dig @10.96.0.10 service-externalname.dev.svc.cluster.local
service-externalname.dev.svc.cluster.local. 30 IN CNAME www.baidu.com.
www.baidu.com.          30      IN      CNAME   www.a.shifen.com.
www.a.shifen.com.       30      IN      A       39.156.66.18
www.a.shifen.com.       30      IN      A       39.156.66.14
```

## 3.Ingress介绍

在前面课程中已经提到，Service 对集群之外暴露服务的主要方式有两种：NotePort 和 LoadBalancer，但是这两种方式，都有一定的缺点：

- NodePort 方式的缺点是会占用很多集群机器的端口，那么当集群服务变多的时候，这个缺点就愈发明显
- LB 方式的缺点是每个 service 需要一个 LB，浪费、麻烦，并且需要 kubernetes 之外设备的支持

基于这种现状，kubernetes 提供了 Ingress 资源对象，Ingress 只需要一个 NodePort 或者一个 LB 就可以满足暴露多个 Service 的需求。工作机制大致如下图表示：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200623092808049.png" alt="img" style="zoom: 80%;" />

实际上，Ingress 相当于一个 7 层的负载均衡器，是 kubernetes 对反向代理的一个抽象，它的工作原理类似于 Nginx，可以理解成在 Ingress 里建立诸多映射规则，Ingress Controller 通过监听这些配置规则并转化成 Nginx 的反向代理配置 , 然后对外部提供服务。在这里有两个核心概念：

- `ingress`：kubernetes 中的一个对象，作用是定义请求如何转发到 service 的规则
- `ingress controller`：具体实现反向代理及负载均衡的程序，对 ingress 定义的规则进行解析，根据配置的规则来实现请求转发，实现方式有很多，比如 Nginx, Contour, Haproxy 等等

Ingress（以 Nginx 为例）的工作原理如下：

1. 用户编写 Ingress 规则，说明哪个域名对应 kubernetes 集群中的哪个 Service
2. Ingress 控制器动态感知 Ingress 服务规则的变化，然后生成一段对应的 Nginx 反向代理配置
3. Ingress 控制器会将生成的 Nginx 配置写入到一个运行着的 Nginx 服务中，并动态更新
4. 到此为止，其实真正在工作的就是一个 Nginx 了，内部配置了用户定义的请求转发规则

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200516112704764.png" alt="img" style="zoom:67%;" />

## 4.Ingress使用

### 4.1 搭建ingress环境

yaml 文件：https://kubernetes.github.io/ingress-nginx/deploy/#quick-start

```bash
# 创建 ingress-nginx
[root@master youyi]# kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/cloud/deploy.yaml

# 等待所有组件安装完成
[root@master youyi]# kubectl wait --namespace ingress-nginx \
>   --for=condition=ready pod \
>   --selector=app.kubernetes.io/component=controller \
>   --timeout=120s
pod/ingress-nginx-controller-65dc77f88f-w6l4v condition met

# 查看 ingress-nginx
[root@master youyi]# kubectl get pod -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-glmmz        0/1     Completed   0          118s
ingress-nginx-admission-patch-6xbj6         0/1     Completed   2          118s
ingress-nginx-controller-65dc77f88f-w6l4v   1/1     Running     0          118s

# 查看 service
[root@master youyi]# kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.96.19.74    <pending>     80:31964/TCP,443:30326/TCP   59m
ingress-nginx-controller-admission   ClusterIP      10.106.42.57   <none>        443/TCP                      59m
```

### 4.2 准备service和pod

为了后面的实验比较方便，创建如下图所示的模型

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200516102419998.png" alt="img" style="zoom: 67%;" />

创建 tomcat-nginx.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat-pod
  template:
    metadata:
      labels:
        app: tomcat-pod
    spec:
      containers:
      - name: tomcat
        image: tomcat:8.5-jre10-slim # 此版本的 tomcat 有默认首页，易于测试查看
        ports:
        - containerPort: 8080

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
  namespace: dev
spec:
  selector:
    app: tomcat-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
```

```bash
# 创建
[root@k8s-master01 ~]# kubectl create -f tomcat-nginx.yaml

# 查看
[root@k8s-master01 ~]# kubectl get svc -n dev
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
nginx-service    ClusterIP   None         <none>        80/TCP     48s
tomcat-service   ClusterIP   None         <none>        8080/TCP   48s
```

### 4.3 Http代理

创建 ingress-http.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-http-nginx
  namespace: dev
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.itheima.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: nginx-service
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-http-tomcat
  namespace: dev
spec:
  ingressClassName: nginx
  rules:
  - host: tomcat.itheima.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: tomcat-service
            port:
              number: 8080
```

> **注意**
>
> 最好为每个 servoce 单独指定 host

```bash
# 创建
[root@k8s-master01 ~]# kubectl create -f ingress-http.yaml
ingress.extensions/ingress-http created

# 查看详情
[root@master youyi]# kubectl get ing -n dev
NAME                  CLASS   HOSTS                ADDRESS   PORTS   AGE
ingress-http-nginx    nginx   nginx.itheima.com              80      17s
ingress-http-tomcat   nginx   tomcat.itheima.com             80      17s

[root@master youyi]# kubectl describe ing ingress-http-nginx -n dev
Name:             ingress-http-nginx
Labels:           <none>
Namespace:        dev
Address:          
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host               Path  Backends
  ----               ----  --------
  nginx.itheima.com  
                     /   nginx-service:80 (10.244.1.39:80,10.244.1.40:80,10.244.2.36:80)
Annotations:         <none>
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  Sync    28s   nginx-ingress-controller  Scheduled for sync
  
[root@master youyi]# kubectl describe ing ingress-http-tomcat -n dev
Name:             ingress-http-tomcat
Labels:           <none>
Namespace:        dev
Address:          
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host                Path  Backends
  ----                ----  --------
  tomcat.itheima.com  
                      /   tomcat-service:8080 (10.244.1.41:8080,10.244.2.34:8080,10.244.2.35:8080)
Annotations:          <none>
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  Sync    42s   nginx-ingress-controller  Scheduled for sync
```

接下来在本地电脑上配置 host 文件，解析上面的两个域名到 192.168.11.100（master）上。然后就可以分别访问 tomcat.itheima.com:31964 和 nginx.itheima.com:31964 查看效果了。

### 4.4 Https代理

> **疑问**
>
> 不用这个代理也能访问

创建证书

```bash
# 生成证书
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/C=CN/ST=BJ/L=BJ/O=nginx/CN=youyi.com"

# 创建密钥
kubectl create secret tls tls-secret --key tls.key --cert tls.crt
```

创建 ingress-https.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-https-nginx
  namespace: dev
spec:
  tls:
    - hosts:
      - nginx.itheima.com
      secretName: tls-secret # 指定秘钥
  rules:
  - host: nginx.itheima.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: nginx-service
            port:
              number: 80

---

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-https-tomcat
  namespace: dev
spec:
  tls:
    - hosts:
      - tomcat.itheima.com
      secretName: tls-secret # 指定秘钥
  rules:
  - host: tomcat.itheima.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: tomcat-service
            port:
              number: 8080
```

```bash
# 创建
[root@k8s-master01 ~]# kubectl create -f ingress-https.yaml
ingress.extensions/ingress-https created

# 查看
[root@k8s-master01 ~]# kubectl get ing -n dev
NAME            HOSTS                                  ADDRESS         PORTS     AGE
ingress-https   nginx.itheima.com,tomcat.itheima.com   10.104.184.38   80, 443   2m42s

# 查看详情
[root@k8s-master01 ~]# kubectl describe ing ingress-https-nginx -n dev
...
TLS:
  tls-secret terminates nginx.itheima.com,tomcat.itheima.com
Rules:
Host              Path Backends
----              ---- --------
nginx.itheima.com  /  nginx-service:80 (10.244.1.97:80,10.244.1.98:80,10.244.2.119:80)
tomcat.itheima.com /  tomcat-service:8080(10.244.1.99:8080,10.244.2.117:8080,10.244.2.120:8080)
...
```

下面可以通过浏览器访问 https://nginx.itheima.com:30326 和 https://tomcat.itheima.com:30326 来查看了