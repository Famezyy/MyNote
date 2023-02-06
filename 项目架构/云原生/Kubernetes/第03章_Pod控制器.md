# 第03章_Pod控制器

## 1.介绍

Pod 是 kubernetes 的最小管理单元，在 kubernetes 中，按照 Pod 的创建方式可以将其分为两类：

- 自主式 Pod：kubernetes直接创建出来的 Pod，这种 Pod 删除后就没有了，也不会重建
- 控制器创建的 Pod：kubernetes 通过控制器创建的 Pod，这种 Pod 删除了之后还会自动重建

> **`什么是 Pod 控制器`**
>
> Pod 控制器是管理 Pod 的中间层，使用 Pod 控制器之后，只需要告诉 Pod 控制器，想要多少个什么样的 Pod 就可以了，它会创建出满足条件的 Pod 并确保每一个 Pod 资源处于用户期望的目标状态。如果 Pod 资源在运行中出现故障，它会基于指定策略重新编排 Pod。

在 kubernetes 中，有很多类型的 Pod 控制器，每种都有自己的适合的场景，常见的有下面这些：

- `ReplicationController`：比较原始的 Pod 控制器，已经被废弃，由`ReplicaSet`替代
- `ReplicaSet`：保证副本数量一直维持在期望值，并支持pod数量扩缩容，镜像版本升级
- `Deployment`：通过控制`ReplicaSet`来控制 Pod，并支持滚动升级、回退版本
- `Horizontal Pod Autoscaler`：可以根据集群负载自动水平调整 Pod 的数量，实现削峰填谷
- `DaemonSet`：在集群中的指定 Node 上运行且仅运行一个副本，一般用于守护进程类的任务，例如收集日志
- `Job`：它创建出来的 Pod 只要完成任务就立即退出，不需要重启或重建，用于执行一次性任务
- `Cronjob`：它创建的 Pod 负责周期性任务控制，不需要持续后台运行
- `StatefulSet`：管理有状态应用

## 2.ReplicaSet(RS)

ReplicaSet 的主要作用是**保证一定数量的 pod 正常运行**，它会持续监听这些 Pod 的运行状态，一旦 Pod 发生故障，就会重启或重建。同时它还支持对 Pod 数量的扩缩容和镜像版本的升降级。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302010049810.png" alt="img"  />

ReplicaSet 的资源清单文件：

```yaml
apiVersion: apps/v1 # 版本号
kind: ReplicaSet # 类型       
metadata: # 元数据
  name: # rs 名称 
  namespace: # 所属命名空间 
  labels: # 标签
    controller: rs
spec: # 详情描述
  replicas: 3 # 副本数量
  selector: # 选择器，通过它指定该控制器管理哪些 Pod
    matchLabels:      # Labels 匹配规则
      app: nginx-pod
    matchExpressions: # Expressions 匹配规则
    - key: app, 
      operator: In,
      values: [nginx-pod]
  template: # 模板，当副本数量不足时，会根据下面的模板创建 Pod 副本
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

在这里面，需要新了解的配置项就是`spec`下面几个选项：

- `replicas`：指定副本数量，其实就是当前 rs 创建出来的 Pod 的数量，默认为 1

- `selector`：选择器，它的作用是建立 Pod 控制器和 Pod 之间的关联关系，采用的 Label Selector 机制

  在 Pod 模板上定义 label，在控制器上定义选择器，就可以表明当前控制器能管理哪些 Pod 了

- `template`：模板，就是当前控制器创建 Pod 所使用的模板板，里面其实就是前一章学过的 Pod 的定义

### 2.1 创建ReplicaSet

创建 pc-replicaset.yaml 文件，内容如下：

```yaml
apiVersion: apps/v1
kind: ReplicaSet   
metadata:
  name: pc-replicaset
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
```

```bash
# 创建 rs
[root@k8s-master01 ~]# kubectl create -f pc-replicaset.yaml
replicaset.apps/pc-replicaset created

# 查看 rs
# DESIRED：期望副本数量  
# CURRENT：当前副本数量  
# READY：已经准备好提供服务的副本数量
[root@k8s-master01 ~]# kubectl get rs pc-replicaset -n dev -o wide
NAME          DESIRED   CURRENT READY AGE   CONTAINERS   IMAGES             SELECTOR
pc-replicaset 3         3       3     22s   nginx        nginx:1.17.1       app=nginx-pod

# 查看当前控制器创建出来的 Pod
# 这里发现控制器创建出来的 Pod 的名称是在控制器名称后面拼接了 -xxxxx 随机码
[root@k8s-master01 ~]# kubectl get pod -n dev
NAME                          READY   STATUS    RESTARTS   AGE
pc-replicaset-6vmvt   1/1     Running   0          54s
pc-replicaset-fmb8f   1/1     Running   0          54s
pc-replicaset-snrk2   1/1     Running   0          54s
```

### 2.2 扩缩容

#### 1.edit命令

```bash
# 编辑 rs 的副本数量，修改 spec:replicas: 6 即可
[root@k8s-master01 ~]# kubectl edit rs pc-replicaset -n dev
replicaset.apps/pc-replicaset edited

# 查看pod
[root@k8s-master01 ~]# kubectl get pods -n dev
NAME                          READY   STATUS    RESTARTS   AGE
pc-replicaset-6vmvt   1/1     Running   0          114m
pc-replicaset-cftnp   1/1     Running   0          10s
pc-replicaset-fjlm6   1/1     Running   0          10s
pc-replicaset-fmb8f   1/1     Running   0          114m
pc-replicaset-s2whj   1/1     Running   0          10s
pc-replicaset-snrk2   1/1     Running   0          114m
```

#### 2.scale命令

```bash
# 使用 scale 命令实现扩缩容， 后面 --replicas=n 直接指定目标数量即可
[root@k8s-master01 ~]# kubectl scale rs pc-replicaset --replicas=2 -n dev
replicaset.apps/pc-replicaset scaled

# 命令运行完毕，立即查看，发现已经有 4 个开始准备退出了
[root@k8s-master01 ~]# kubectl get pods -n dev
NAME                       READY   STATUS        RESTARTS   AGE
pc-replicaset-6vmvt   0/1     Terminating   0          118m
pc-replicaset-cftnp   0/1     Terminating   0          4m17s
pc-replicaset-fjlm6   0/1     Terminating   0          4m17s
pc-replicaset-fmb8f   1/1     Running       0          118m
pc-replicaset-s2whj   0/1     Terminating   0          4m17s
pc-replicaset-snrk2   1/1     Running       0          118m

# 稍等片刻，就只剩下 2 个了
[root@k8s-master01 ~]# kubectl get pods -n dev
NAME                       READY   STATUS    RESTARTS   AGE
pc-replicaset-fmb8f   1/1     Running   0          119m
pc-replicaset-snrk2   1/1     Running   0          119m
```

### 2.3 镜像升级

```bash
# 编辑 rs 的容器镜像 - image: nginx:1.17.2
[root@k8s-master01 ~]# kubectl edit rs pc-replicaset -n dev
replicaset.apps/pc-replicaset edited

# 再次查看，发现镜像版本已经变更了
[root@k8s-master01 ~]# kubectl get rs -n dev -o wide
NAME                DESIRED  CURRENT   READY   AGE    CONTAINERS   IMAGES        ...
pc-replicaset       2        2         2       140m   nginx         nginx:1.17.2  ...

# 同样的道理，也可以使用命令完成这个工作
# kubectl set image rs rs名称 容器=镜像版本 -n namespace
[root@k8s-master01 ~]# kubectl set image rs pc-replicaset nginx=nginx:1.17.1  -n dev
replicaset.apps/pc-replicaset image updated

# 再次查看，发现镜像版本已经变更了
[root@k8s-master01 ~]# kubectl get rs -n dev -o wide
NAME                 DESIRED  CURRENT   READY   AGE    CONTAINERS   IMAGES            ...
pc-replicaset        2        2         2       145m   nginx        nginx:1.17.1 ... 
```

> **注意**
>
> 升级仅对之后创建的容器有效，现有容器版本不变。

### 2.4 删除ReplicaSet

```bash
# 使用 kubectl delete 命令会删除此 RS 以及它管理的 Pod
# 在 kubernetes 删除 RS 前，会将 RS 的 replicasclear 调整为 0，等待所有的 Pod 被删除后，在执行 RS 对象的删除
[root@k8s-master01 ~]# kubectl delete rs pc-replicaset -n dev
replicaset.apps "pc-replicaset" deleted
[root@k8s-master01 ~]# kubectl get pod -n dev -o wide
No resources found in dev namespace.

# 如果希望仅仅删除 RS 对象（保留 Pod），可以使用 kubectl delete 命令时添加 --cascade=false 选项（不推荐）
[root@k8s-master01 ~]# kubectl delete rs pc-replicaset -n dev --cascade=false
replicaset.apps "pc-replicaset" deleted
[root@k8s-master01 ~]# kubectl get pods -n dev
NAME                  READY   STATUS    RESTARTS   AGE
pc-replicaset-cl82j   1/1     Running   0          75s
pc-replicaset-dslhb   1/1     Running   0          75s

# 也可以使用 yaml 直接删除（推荐）
[root@k8s-master01 ~]# kubectl delete -f pc-replicaset.yaml
replicaset.apps "pc-replicaset" deleted
```

## 3.Deployment(Deploy)

为了更好的解决服务编排的问题，kubernetes 在 V1.2 版本开始，引入了 Deployment 控制器。值得一提的是，这种控制器并不直接管理 Pod，而是通过管理 ReplicaSet 来简介管理 Pod。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302010049470.png" alt="img" style="zoom:80%;" />

Deployment 主要功能有下面几个：

- 支持 ReplicaSet 的所有功能
- 支持发布的停止、继续
- 支持滚动升级和回滚版本

Deployment 的资源清单文件：

```yaml
apiVersion: apps/v1 # 版本号
kind: Deployment # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: deploy
spec: # 详情描述
  replicas: 3 # 副本数量
  revisionHistoryLimit: 3 # 保留的历史版本上限
  paused: false # 暂停部署，即创建 Deployment 后是否要暂停而不是立即部署，默认是 false，即立即部署
  progressDeadlineSeconds: 600 # 部署超时时间（s），默认是 600
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxSurge: 30% # 最大额外可以存在的副本数，可以为百分比，也可以为整数
      maxUnavailable: 30% # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理哪些 Pod
    matchLabels:      # Labels 匹配规则
      app: nginx-pod
    matchExpressions: # Expressions 匹配规则
    - key: app,
      operator: In,
      values: [nginx-pod]
  template: # 模板，当副本数量不足时，会根据下面的模板创建 Pod 副本
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

### 3.1 创建Deployment

创建 pc-deployment.yaml，内容如下：

```
apiVersion: apps/v1
kind: Deployment      
metadata:
  name: pc-deployment
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
```

可以发现创建了相应的 pod，同时也创建了 RS：

```bash
[root@master ~]# kubectl get deploy -n dev -o wide
NAME            READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
pc-deployment   3/3     3            3           13s   nginx        nginx:1.17.1   app=nginx-pod

[root@master ~]# kubectl get pod -n dev -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED NODE   READINESS GATES
pc-deployment-84c7dcff9-9hfs4   1/1     Running   0          24s   10.244.2.21   node2   <none>           <none>
pc-deployment-84c7dcff9-c77w6   1/1     Running   0          24s   10.244.2.20   node2   <none>           <none>
pc-deployment-84c7dcff9-pkc8q   1/1     Running   0          24s   10.244.2.19   node2   <none>           <none>

[root@master ~]# kubectl get rs -n dev -o wide
NAME                      DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES         SELECTOR
pc-deployment-84c7dcff9   3         3         3       48s   nginx        nginx:1.17.1   app=nginx-pod,pod-template-hash=84c7dcff9
```

### 3.2 扩缩容

`Deployment`的扩缩容与`ReplicaSet`一致。

#### 1.edit命令

```bash
# 编辑 deployment 的副本数量，修改 spec:replicas: 4 即可
[root@k8s-master01 ~]# kubectl edit deploy pc-deployment -n dev
deployment.apps/pc-deployment edited

# 查看 pod
[root@k8s-master01 ~]# kubectl get pods -n dev
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-6696798b78-d2c8n   1/1     Running   0          5m23s
pc-deployment-6696798b78-jxmdq   1/1     Running   0          2m38s
pc-deployment-6696798b78-smpvp   1/1     Running   0          5m23s
pc-deployment-6696798b78-wvjd8   1/1     Running   0          5m23s
```

#### 2.scale命令

```
# 变更副本数量为 5 个
[root@k8s-master01 ~]# kubectl scale deploy pc-deployment --replicas=5 -n dev
deployment.apps/pc-deployment scaled

# 查看 deployment
[root@k8s-master01 ~]# kubectl get deploy pc-deployment -n dev
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
pc-deployment   5/5     5            5           2m

# 查看 pod
[root@k8s-master01 ~]#  kubectl get pods -n dev
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-6696798b78-d2c8n   1/1     Running   0          4m19s
pc-deployment-6696798b78-jxmdq   1/1     Running   0          94s
pc-deployment-6696798b78-mktqv   1/1     Running   0          93s
pc-deployment-6696798b78-smpvp   1/1     Running   0          4m19s
pc-deployment-6696798b78-wvjd8   1/1     Running   0          4m19s
```

### 3.3 镜像更新策略

deployment 支持两种更新策略:`重建更新`和`滚动更新（默认）`，可以通过`strategy`指定策略类型。

`strategy`：指定新的 Pod 替换旧的 Pod 的策略，支持两个属性

- `type`：指定策略类型，支持两种策略

  - `Recreate`：在创建出新的 Pod 之前会先杀掉所有已存在的 Pod
  - `RollingUpdate`：滚动更新，就是杀死一部分，就启动一部分，在更新过程中，存在两个版本 Pod

- `rollingUpdate`：当`type`为`RollingUpdate`时生效，用于为`RollingUpdate`设置参数，支持两个属性

  - `maxUnavailable`：用来指定在升级过程中不可用 Pod 的最大数量，默认为 25%
  - `maxSurge`：用来指定在升级过程中可以超过期望的 Pod 的最大数量，默认为 25%

  > **提示**
  >
  > 因为在滚动更新过程中是新增一个新版本的 Pod，删除一个老版本的 Pod，所以可以指定新增 Pod 的话 Pod 的数量不能超过多少，删除一个 Pod 的话最多能删多少

#### 1.重建更新

编辑 pc-deployment.yaml，在`spec`节点下添加更新策略

```yaml
apiVersion: apps/v1
kind: Deployment      
metadata:
  name: pc-deployment
  namespace: dev
spec: 
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  strategy: # 策略
    type: Recreate # 重建更新
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
```

创建 deploy 进行验证

```bash
# 更新 Deployment
[root@master ~]# kubectl apply -f pc-deployment.yaml 
deployment.apps/pc-deployment configured

# 变更镜像
[root@k8s-master01 ~]# kubectl set image deployment pc-deployment nginx=nginx:1.17.2 -n dev
deployment.apps/pc-deployment image updated

# 观察升级过程
[root@k8s-master01 ~]#  kubectl get pods -n dev -w
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-5d89bdfbf9-65qcw   1/1     Running   0          31s
pc-deployment-5d89bdfbf9-w5nzv   1/1     Running   0          31s
pc-deployment-5d89bdfbf9-xpt7w   1/1     Running   0          31s

pc-deployment-5d89bdfbf9-xpt7w   1/1     Terminating   0          41s
pc-deployment-5d89bdfbf9-65qcw   1/1     Terminating   0          41s
pc-deployment-5d89bdfbf9-w5nzv   1/1     Terminating   0          41s

pc-deployment-675d469f8b-grn8z   0/1     Pending       0          0s
pc-deployment-675d469f8b-hbl4v   0/1     Pending       0          0s
pc-deployment-675d469f8b-67nz2   0/1     Pending       0          0s

pc-deployment-675d469f8b-grn8z   0/1     ContainerCreating   0          0s
pc-deployment-675d469f8b-hbl4v   0/1     ContainerCreating   0          0s
pc-deployment-675d469f8b-67nz2   0/1     ContainerCreating   0          0s

pc-deployment-675d469f8b-grn8z   1/1     Running             0          1s
pc-deployment-675d469f8b-67nz2   1/1     Running             0          1s
pc-deployment-675d469f8b-hbl4v   1/1     Running             0          2s
```

#### 2.滚动更新

编辑pc-deployment.yaml，在`spec`节点下添加更新策略

```
spec:
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate:
      maxSurge: 25% 
      maxUnavailable: 25%
```

创建 deploy 进行验证

```bash
# 更新 Deployment
[root@master ~]# kubectl apply -f pc-deployment.yaml 
deployment.apps/pc-deployment configured

# 变更镜像
[root@k8s-master01 ~]# kubectl set image deployment pc-deployment nginx=nginx:1.17.3 -n dev 
deployment.apps/pc-deployment image updated

# 观察升级过程　-w：阻塞显示界面
[root@k8s-master01 ~]# kubectl get pods -n dev -w
NAME                           READY   STATUS    RESTARTS   AGE
pc-deployment-c848d767-8rbzt   1/1     Running   0          31m
pc-deployment-c848d767-h4p68   1/1     Running   0          31m
pc-deployment-c848d767-hlmz4   1/1     Running   0          31m
pc-deployment-c848d767-rrqcn   1/1     Running   0          31m

pc-deployment-966bf7f44-226rx   0/1     Pending             0          0s
pc-deployment-966bf7f44-226rx   0/1     ContainerCreating   0          0s
pc-deployment-966bf7f44-226rx   1/1     Running             0          1s
pc-deployment-c848d767-h4p68    0/1     Terminating         0          34m

pc-deployment-966bf7f44-cnd44   0/1     Pending             0          0s
pc-deployment-966bf7f44-cnd44   0/1     ContainerCreating   0          0s
pc-deployment-966bf7f44-cnd44   1/1     Running             0          2s
pc-deployment-c848d767-hlmz4    0/1     Terminating         0          34m

pc-deployment-966bf7f44-px48p   0/1     Pending             0          0s
pc-deployment-966bf7f44-px48p   0/1     ContainerCreating   0          0s
pc-deployment-966bf7f44-px48p   1/1     Running             0          0s
pc-deployment-c848d767-8rbzt    0/1     Terminating         0          34m

pc-deployment-966bf7f44-dkmqp   0/1     Pending             0          0s
pc-deployment-966bf7f44-dkmqp   0/1     ContainerCreating   0          0s
pc-deployment-966bf7f44-dkmqp   1/1     Running             0          2s
pc-deployment-c848d767-rrqcn    0/1     Terminating         0          34m

# 至此，新版本的 pod 创建完毕，旧版本的 pod 销毁完毕
# 中间过程是滚动进行的，也就是边销毁边创建
```

滚动更新的过程：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202302010049338.png" alt="img" style="zoom:67%;" />

镜像更新后 rs 的变化

```
# 查看 rs，发现原来的 rs 依旧存在，只是 Pod 数量变为了 0，最后在运行的 rs 的 Pod 数量为 4
# 其实这就是 deployment 能够进行版本回退的关键
[root@k8s-master01 ~]# kubectl get rs -n dev
NAME                       DESIRED   CURRENT   READY   AGE
pc-deployment-6696798b78   0         0         0       7m37s
pc-deployment-6696798b11   0         0         0       5m37s
pc-deployment-c848d76789   4         4         4       72s
```

### 3.3 版本回退

Deployment 支持版本升级过程中的暂停、继续功能以及版本回退等诸多功能。

`kubectl rollout`是进行版本升级相关功能的命令，支持下面的选项：

- `status`：显示当前升级状态
- `history`：显示升级历史记录
- `pause`：暂停版本升级过程
- `resume`：继续已经暂停的版本升级过程
- `restart`：重启版本升级过程
- `undo`：回滚到上一级版本（可以使用`--to-revision`回滚到指定版本）

```bash
# 查看当前升级版本的状态
[root@master youyi]# kubectl rollout status deploy pc-deployment -n dev
deployment "pc-deployment" successfully rolled out

# 查看升级历史记录
[root@k8s-master01 ~]# kubectl rollout history deploy pc-deployment -n dev
deployment.apps/pc-deployment
REVISION  CHANGE-CAUSE
1         kubectl create --filename=pc-deployment.yaml --record=true
2         kubectl create --filename=pc-deployment.yaml --record=true
3         kubectl create --filename=pc-deployment.yaml --record=true
# 可以发现有三次版本记录，说明完成过两次升级

# 版本回滚
# 这里直接使用 --to-revision=1 回滚到了 1 版本，如果省略这个选项，就是回退到上个版本，就是 2 版本
[root@master youyi]# kubectl rollout undo deploy pc-deployment --to-revision=1 -n dev
deployment.apps/pc-deployment rolled back

# 查看发现，通过 nginx 镜像版本可以发现到了第一版
[root@master youyi]# kubectl get deploy -n dev -o wide
NAME            READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES         SELECTOR
pc-deployment   3/3     3            3           3m6s   nginx        nginx:1.17.1   app=nginx-pod

# 查看 rs，发现第一个 rs 中有 3 个 pod 运行，后面两个版本的 rs 中 pod 未运行
# 其实 deployment 之所以可是实现版本的回滚，就是通过记录下历史 rs 来实现的
# 一旦想回滚到哪个版本，只需要将当前版本 pod 数量降为 0，然后将回滚版本的 pod 提升为目标数量就可以了
[root@master youyi]# kubectl get rs -n dev -o wide
NAME                      DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES         SELECTOR
pc-deployment-84c7dcff9   3         3         3       3m10s   nginx        nginx:1.17.1   app=nginx-pod,pod-template-hash=84c7dcff9
pc-deployment-d8657876f   0         0         0       3m4s    nginx        nginx:1.17.3   app=nginx-pod,pod-template-hash=d8657876f
pc-deployment-dd8674d76   0         0         0       2m5s    nginx        nginx:1.17.4   app=nginx-pod,pod-template-hash=dd8674d76
```

### 3.4 金丝雀发布

Deployment 控制器支持控制更新过程中的控制，如“暂停（pause）”或“继续（resume）”更新操作。

比如有一批新的 Pod 资源创建完成后立即暂停更新过程，此时，仅存在一部分新版本的应用，主体部分还是旧的版本。然后，再筛选一小部分的用户请求路由到新版本的 Pod 应用，继续观察能否稳定地按期望的方式运行。确定没问题之后再继续完成余下的 Pod 资源滚动更新，否则立即回滚更新操作。这就是所谓的金丝雀发布。

```
# 更新 deployment 的版本，并配置暂停 deployment
[root@k8s-master01 ~]# kubectl set image deploy pc-deployment nginx=nginx:1.17.4 -n dev && kubectl rollout pause deployment pc-deployment -n dev
deployment.apps/pc-deployment image updated
deployment.apps/pc-deployment paused

# 观察更新状态
[root@k8s-master01 ~]# kubectl rollout status deploy pc-deployment -n dev　
Waiting for deployment "pc-deployment" rollout to finish: 2 out of 4 new replicas have been updated...

# 监控更新的过程，可以看到已经新增了一个资源，但是并未按照预期的状态去删除一个旧的资源，就是因为使用了 pause 暂停命令
[root@k8s-master01 ~]# kubectl get rs -n dev -o wide
NAME                       DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES         
pc-deployment-5d89bdfbf9   3         3         3       19m     nginx        nginx:1.17.1   
pc-deployment-675d469f8b   0         0         0       14m     nginx        nginx:1.17.2   
pc-deployment-6c9f56fcfb   2         2         2       3m16s   nginx        nginx:1.17.4   
[root@k8s-master01 ~]# kubectl get pods -n dev
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-5d89bdfbf9-rj8sq   1/1     Running   0          7m33s
pc-deployment-5d89bdfbf9-ttwgg   1/1     Running   0          7m35s
pc-deployment-5d89bdfbf9-v4wvc   1/1     Running   0          7m34s
pc-deployment-6c9f56fcfb-996rt   1/1     Running   0          3m31s
pc-deployment-6c9f56fcfb-j2gtj   1/1     Running   0          3m31s

# 确保更新的 pod 没问题了，继续更新
[root@k8s-master01 ~]# kubectl rollout resume deploy pc-deployment -n dev
deployment.apps/pc-deployment resumed

# 出现问题时，先 resume 再 undo
[root@master youyi]# kubectl rollout resume deploy pc-deployment -n dev && kubectl rollout undo deploy pc-deployment -n dev

# 查看最后的更新情况
[root@k8s-master01 ~]# kubectl get rs -n dev -o wide
NAME                       DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES         
pc-deployment-5d89bdfbf9   0         0         0       21m     nginx        nginx:1.17.1   
pc-deployment-675d469f8b   0         0         0       16m     nginx        nginx:1.17.2   
pc-deployment-6c9f56fcfb   4         4         4       5m11s   nginx        nginx:1.17.4   

[root@k8s-master01 ~]# kubectl get pods -n dev
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-6c9f56fcfb-7bfwh   1/1     Running   0          37s
pc-deployment-6c9f56fcfb-996rt   1/1     Running   0          5m27s
pc-deployment-6c9f56fcfb-j2gtj   1/1     Running   0          5m27s
pc-deployment-6c9f56fcfb-rf84v   1/1     Running   0          37s
```

**删除Deployment**

```
# 删除 deployment，其下的 rs 和 pod 也将被删除
[root@k8s-master01 ~]# kubectl delete -f pc-deployment.yaml
deployment.apps "pc-deployment" deleted
```

## 4.Horizontal Pod Autoscaler(HPA)

在前面的课程中，我们已经可以实现通过手工执行`kubectl scale`命令实现 Pod 扩容或缩容，但是这显然不符合 Kubernetes 的定位目标：自动化、智能化。 Kubernetes 期望可以实现通过监测 Pod 的使用情况，实现 Pod 数量的自动调整，于是就产生了 Horizontal Pod Autoscaler（HPA）这种控制器。

HPA 可以获取每个 Pod 利用率，然后和 HPA 中定义的指标进行对比，同时计算出需要伸缩的具体值，最后实现 Pod 的数量的调整。其实 HPA 与之前的 Deployment 一样，也属于一种 Kubernetes 资源对象，它通过追踪分析 RC 控制的所有目标 Pod 的负载变化情况，来确定是否需要针对性地调整目标 Pod 的副本数，这是 HPA 的实现原理。HPA 是通过控制 Deployment，由 Deployment 进一步控制 ReplicaSet 来对 Pod 进行管理。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200608155858271.png" alt="img" style="zoom: 67%;" />

接下来，我们来做一个实验

### 4.1 安装metrics-server

官网：https://github.com/kubernetes-sigs/metrics-server/blob/master/README.md

`metrics-server`可以用来收集集群中的资源使用情况

```
# 安装
[root@k8s-master01 ~]# wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 修改 deployment, 注意修改的是镜像和初始化参数
[root@k8s-master01 ~]# vim components.yaml
```

添加下面选项

```yaml
spec:
  hostNetwork: true
  containers:
  # - image: registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server-amd64:v0.3.6
    args:
    - --kubelet-insecure-tls
  #  - --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP
```

```bash
# 安装 metrics-server
[root@k8s-master01 1.8+]# kubectl create -f components.yaml

# 查看 pod 运行情况
[root@k8s-master01 1.8+]# kubectl get pod -n kube-system
metrics-server-6b976979db-2xwbj   1/1     Running   0          90s

# 使用 kubectl top node 查看资源使用情况（安装后需等待一段时间用于采集信息）
[root@k8s-master01 1.8+]# kubectl top node
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master01   289m         14%    1582Mi          54%       
k8s-node01     81m          4%     1195Mi          40%       
k8s-node02     72m          3%     1211Mi          41% 

[root@k8s-master01 1.8+]# kubectl top pod -n kube-system
NAME                              CPU(cores)   MEMORY(bytes)
coredns-6955765f44-7ptsb          3m           9Mi
coredns-6955765f44-vcwr5          3m           8Mi
etcd-master                       14m          145Mi
...
```

至此，`metrics-server`安装完成。

### 4.2 准备deployment和servie

创建 pc-hpa-pod.yaml 文件，内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: dev
spec:
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
  replicas: 1
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
        resources: # 资源配额
          limits:  # 限制资源（上限）
            cpu: "1" # CPU 限制，单位是 core 数
          requests: # 请求资源（下限）
            cpu: "100m"  # CPU 限制，单位是 core 数
```

```bash
# 创建 deployment
[root@k8s-master01 1.8+]# kubectl create -f pc-hpa-pod.yaml

# 创建 service
[root@k8s-master01 1.8+]# kubectl expose deployment nginx --type=NodePort --port=80 -n dev

# 查看
[root@k8s-master01 1.8+]# kubectl get deployment,pod,svc -n dev
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           47s

NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-7df9756ccc-bh8dr   1/1     Running   0          47s

NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/nginx   NodePort   10.101.18.29   <none>        80:31830/TCP   35s
```

### 4.3 部署HPA

创建 pc-hpa.yaml 文件，内容如下：

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: pc-hpa
  namespace: dev
spec:
  minReplicas: 1  #最小 pod 数量
  maxReplicas: 10 #最大 pod 数量
  targetCPUUtilizationPercentage: 3 # CPU 使用率指标（%）
  scaleTargetRef:   # 指定要控制的 nginx 信息
    apiVersion:  apps/v1
    kind: Deployment
    name: nginx
```

```bash
# 创建 hpa
[root@k8s-master01 1.8+]# kubectl create -f pc-hpa.yaml
horizontalpodautoscaler.autoscaling/pc-hpa created

# 查看 hpa
[root@k8s-master01 1.8+]# kubectl get hpa -n dev
NAME     REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
pc-hpa   Deployment/nginx   0%/3%     1         10        1          62s
```

### 4.4 测试

使用压测工具对 service 地址`192.168.5.4:31830`进行压测，然后通过控制台查看 hpa 和 pod 的变化

hpa 变化

```
[root@k8s-master01 ~]# kubectl get hpa -n dev -w
NAME   REFERENCE      TARGETS  MINPODS  MAXPODS  REPLICAS  AGE
pc-hpa  Deployment/nginx  0%/3%   1     10     1      4m11s
pc-hpa  Deployment/nginx  0%/3%   1     10     1      5m19s
pc-hpa  Deployment/nginx  22%/3%   1     10     1      6m50s
pc-hpa  Deployment/nginx  22%/3%   1     10     4      7m5s
pc-hpa  Deployment/nginx  22%/3%   1     10     8      7m21s
pc-hpa  Deployment/nginx  6%/3%   1     10     8      7m51s
pc-hpa  Deployment/nginx  0%/3%   1     10     8      9m6s
pc-hpa  Deployment/nginx  0%/3%   1     10     8      13m
pc-hpa  Deployment/nginx  0%/3%   1     10     1      14m
```

deployment 变化

```
[root@k8s-master01 ~]# kubectl get deployment -n dev -w
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           11m
nginx   1/4     1            1           13m
nginx   1/4     1            1           13m
nginx   1/4     1            1           13m
nginx   1/4     4            1           13m
nginx   1/8     4            1           14m
nginx   1/8     4            1           14m
nginx   1/8     4            1           14m
nginx   1/8     8            1           14m
nginx   2/8     8            2           14m
nginx   3/8     8            3           14m
nginx   4/8     8            4           14m
nginx   5/8     8            5           14m
nginx   6/8     8            6           14m
nginx   7/8     8            7           14m
nginx   8/8     8            8           15m
nginx   8/1     8            8           20m
nginx   8/1     8            8           20m
nginx   1/1     1            1           20m
```

pod 变化

```
[root@k8s-master01 ~]# kubectl get pods -n dev -w
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7df9756ccc-bh8dr   1/1     Running   0          11m
nginx-7df9756ccc-cpgrv   0/1     Pending   0          0s
nginx-7df9756ccc-8zhwk   0/1     Pending   0          0s
nginx-7df9756ccc-rr9bn   0/1     Pending   0          0s
nginx-7df9756ccc-cpgrv   0/1     ContainerCreating   0          0s
nginx-7df9756ccc-8zhwk   0/1     ContainerCreating   0          0s
nginx-7df9756ccc-rr9bn   0/1     ContainerCreating   0          0s
nginx-7df9756ccc-m9gsj   0/1     Pending             0          0s
nginx-7df9756ccc-g56qb   0/1     Pending             0          0s
nginx-7df9756ccc-sl9c6   0/1     Pending             0          0s
nginx-7df9756ccc-fgst7   0/1     Pending             0          0s
nginx-7df9756ccc-g56qb   0/1     ContainerCreating   0          0s
nginx-7df9756ccc-m9gsj   0/1     ContainerCreating   0          0s
nginx-7df9756ccc-sl9c6   0/1     ContainerCreating   0          0s
nginx-7df9756ccc-fgst7   0/1     ContainerCreating   0          0s
nginx-7df9756ccc-8zhwk   1/1     Running             0          19s
nginx-7df9756ccc-rr9bn   1/1     Running             0          30s
nginx-7df9756ccc-m9gsj   1/1     Running             0          21s
nginx-7df9756ccc-cpgrv   1/1     Running             0          47s
nginx-7df9756ccc-sl9c6   1/1     Running             0          33s
nginx-7df9756ccc-g56qb   1/1     Running             0          48s
nginx-7df9756ccc-fgst7   1/1     Running             0          66s
nginx-7df9756ccc-fgst7   1/1     Terminating         0          6m50s
nginx-7df9756ccc-8zhwk   1/1     Terminating         0          7m5s
nginx-7df9756ccc-cpgrv   1/1     Terminating         0          7m5s
nginx-7df9756ccc-g56qb   1/1     Terminating         0          6m50s
nginx-7df9756ccc-rr9bn   1/1     Terminating         0          7m5s
nginx-7df9756ccc-m9gsj   1/1     Terminating         0          6m50s
nginx-7df9756ccc-sl9c6   1/1     Terminating         0          6m50s
```

## 5.DaemonSet(DS)

DaemonSet 类型的控制器可以保证在集群中的每一台（或指定）节点上都运行一个副本。一般适用于日志收集、节点监控等场景。也就是说，如果一个 Pod 提供的功能是节点级别的（每个节点都需要且只需要一个），那么这类 Pod 就适合使用 DaemonSet 类型的控制器创建。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200612010223537.png" alt="img" style="zoom:80%;" />

DaemonSet 控制器的特点：

- 每当向集群中添加一个节点时，指定的 Pod 副本也将添加到该节点上
- 当节点从集群中移除时，Pod 也就被垃圾回收了

下面先来看下 DaemonSet 的资源清单文件

```yaml
apiVersion: apps/v1 # 版本号
kind: DaemonSet # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: daemonset
spec: # 详情描述
  revisionHistoryLimit: 3 # 保留历史版本
  updateStrategy: # 更新策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxUnavailable: 1 # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels　匹配规则
      app: nginx-pod
    matchExpressions: # Expressions　匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建　pod　副本
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

创建 pc-daemonset.yaml，内容如下：

```yaml
apiVersion: apps/v1
kind: DaemonSet      
metadata:
  name: pc-daemonset
  namespace: dev
spec: 
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
```

```bash
# 创建 daemonset
[root@k8s-master01 ~]# kubectl create -f  pc-daemonset.yaml
daemonset.apps/pc-daemonset created

# 查看 daemonset
[root@k8s-master01 ~]#  kubectl get ds -n dev -o wide
NAME        DESIRED  CURRENT  READY  UP-TO-DATE  AVAILABLE   AGE   CONTAINERS   IMAGES         
pc-daemonset   2        2        2      2           2        24s   nginx        nginx:1.17.1   

# 查看 pod，发现在每个 Node 上都运行一个 pod
[root@k8s-master01 ~]#  kubectl get pods -n dev -o wide
NAME                 READY   STATUS    RESTARTS   AGE   IP            NODE    
pc-daemonset-9bck8   1/1     Running   0          37s   10.244.1.43   node1     
pc-daemonset-k224w   1/1     Running   0          37s   10.244.2.74   node2      

# 删除 daemonset
[root@k8s-master01 ~]# kubectl delete -f pc-daemonset.yaml
daemonset.apps "pc-daemonset" deleted
```

## 6.Job

Job，主要用于负责**批量处理（一次要处理指定数量任务）**短暂的**一次性（每个任务仅运行一次就结束）**任务。

Job 特点如下：

- 当 Job 创建的 pod 执行成功结束时，Job 将记录成功结束的 pod 数量
- 当成功结束的 pod 达到指定的数量时，Job 将完成执行

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200618213054113.png" alt="img" style="zoom:80%;" />

Job 的资源清单文件：

```yaml
apiVersion: batch/v1 # 版本号
kind: Job # 类型       
metadata: # 元数据
  name: # rs 名称 
  namespace: # 所属命名空间 
  labels: # 标签
    controller: job
spec: # 详情描述
  completions: 1 # 指定 job 需要成功运行 Pods 的次数。默认值: 1
  parallelism: 1 # 指定 job 在任一时刻应该并发运行 Pods 的数量。默认值: 1
  activeDeadlineSeconds: 30 # 指定 job 可运行的时间期限，超过时间还未结束，系统将会尝试进行终止
  backoffLimit: 6 # 指定 job 失败后进行重试的次数。默认是 6
  manualSelector: true # 是否可以使用 selector 选择器选择 pod，默认是 false
  selector: # 选择器，通过它指定该控制器管理哪些 pod
    matchLabels:      # Labels 匹配规则
      app: counter-pod
    matchExpressions: # Expressions 匹配规则
      - {key: app, operator: In, values: [counter-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建 pod 副本
    metadata:
      labels:
        app: counter-pod
    spec:
      restartPolicy: Never # 重启策略只能设置为 Never 或者 OnFailure
      containers:
      - name: counter
        image: busybox:1.30
        command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 2;done"]
```

> **关于重启策略设置的说明**
>
> - 如果指定为`OnFailure`，则 job 会在 pod 出现故障时重启容器，而不是创建 pod，failed 次数不变
> - 如果指定为`Never`，则 job 会在 pod 出现故障时创建新的 pod，并且故障 pod 不会消失，也不会重启，failed 次数加 1
> - 如果指定为`Always`的话，就意味着一直重启，意味着 job 任务会重复去执行了，当然不对，所以不能设置为`Always`

创建 pc-job.yaml，内容如下：

```yaml
apiVersion: batch/v1
kind: Job      
metadata:
  name: pc-job
  namespace: dev
spec:
  manualSelector: true
  selector:
    matchLabels:
      app: counter-pod
  template:
    metadata:
      labels:
        app: counter-pod
    spec:
      restartPolicy: Never
      containers:
      - name: counter
        image: busybox:1.30
        command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 3;done"]
```

```bash
# 创建 job
[root@k8s-master01 ~]# kubectl create -f pc-job.yaml
job.batch/pc-job created

# 查看 job
[root@k8s-master01 ~]# kubectl get job -n dev -o wide  -w
NAME     COMPLETIONS   DURATION   AGE   CONTAINERS   IMAGES         SELECTOR
pc-job   0/1           21s        21s   counter      busybox:1.30   app=counter-pod
pc-job   1/1           31s        79s   counter      busybox:1.30   app=counter-pod

# 通过观察 pod 状态可以看到，pod 在运行完毕任务后，就会变成 Completed 状态
[root@k8s-master01 ~]# kubectl get pods -n dev -w
NAME           READY   STATUS     RESTARTS      AGE
pc-job-rxg96   1/1     Running     0            29s
pc-job-rxg96   0/1     Completed   0            33s

# 接下来，调整下 pod 运行的总数量和并行数量 即：在 spec 下设置下面两个选项
#  completions: 6 # 指定 job 需要成功运行 Pods 的次数为 6
#  parallelism: 3 # 指定 job 并发运行 Pods 的数量为 3
#  然后重新运行 job，观察效果，此时会发现，job 会每次运行 3 个 pod，总共执行了 6 个 pod
[root@k8s-master01 ~]# kubectl get pods -n dev -w
NAME           READY   STATUS    RESTARTS   AGE
pc-job-684ft   1/1     Running   0          5s
pc-job-jhj49   1/1     Running   0          5s
pc-job-pfcvh   1/1     Running   0          5s
pc-job-684ft   0/1     Completed   0          11s
pc-job-v7rhr   0/1     Pending     0          0s
pc-job-v7rhr   0/1     Pending     0          0s
pc-job-v7rhr   0/1     ContainerCreating   0          0s
pc-job-jhj49   0/1     Completed           0          11s
pc-job-fhwf7   0/1     Pending             0          0s
pc-job-fhwf7   0/1     Pending             0          0s
pc-job-pfcvh   0/1     Completed           0          11s
pc-job-5vg2j   0/1     Pending             0          0s
pc-job-fhwf7   0/1     ContainerCreating   0          0s
pc-job-5vg2j   0/1     Pending             0          0s
pc-job-5vg2j   0/1     ContainerCreating   0          0s
pc-job-fhwf7   1/1     Running             0          2s
pc-job-v7rhr   1/1     Running             0          2s
pc-job-5vg2j   1/1     Running             0          3s
pc-job-fhwf7   0/1     Completed           0          12s
pc-job-v7rhr   0/1     Completed           0          12s
pc-job-5vg2j   0/1     Completed           0          12s

# 删除 job
[root@k8s-master01 ~]# kubectl delete -f pc-job.yaml
job.batch "pc-job" deleted
```

## 7.CronJob(CJ)

CronJob 控制器以 Job 控制器资源为其管控对象，并借助它管理 pod 资源对象，Job 控制器定义的作业任务在其控制器资源创建之后便会立即执行，但**CronJob可以在特定的时间点（反复的）去运行 job 任务**。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20200618213149531.png" alt="img" style="zoom:80%;" />

CronJob 的资源清单文件：

```yaml
apiVersion: batch/v1beta1 # 版本号
kind: CronJob # 类型       
metadata: # 元数据
  name: # rs 名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: cronjob
spec: # 详情描述
  schedule: # cron 格式的作业调度运行时间点,用于控制任务在什么时间执行
  concurrencyPolicy: # 并发执行策略，用于定义前一次作业运行尚未完成时是否以及如何运行后一次的作业
  failedJobHistoryLimit: # 为失败的任务执行保留的历史记录数，默认为 1
  successfulJobHistoryLimit: # 为成功的任务执行保留的历史记录数，默认为 3
  startingDeadlineSeconds: # 启动作业错误的超时时长
  jobTemplate: # job 控制器模板，用于为 cronjob 控制器生成 job 对象;下面其实就是 job 的定义
    metadata:
    spec:
      activeDeadlineSeconds: 30
      manualSelector: true
      selector:
        matchLabels:
          app: counter-pod
      template:
        metadata:
          labels:
            app: counter-pod
        spec:
          restartPolicy: Never 
          containers:
          - name: counter
            image: busybox:1.30
            command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 20;done"]
```

> **需要重点解释的几个选项**
>
> - `schedule`: cron 表达式，用于指定任务的执行时间
>       */1    *      *    *     *
>       <分钟> <小时> <日> <月份> <星期>
>
>   分钟 值从 0 到 59.
>   小时 值从 0 到 23.
>   日 值从 1 到 31.
>   月 值从 1 到 12.
>   星期 值从 0 到 6, 0 代表星期日
>   多个时间可以用逗号隔开；范围可以用连字符给出；*可以作为通配符； /表示每...
>
> - `concurrencyPolicy`:
>   
>   - `Allow`：允许 Jobs 并发运行（默认）
>   - `Forbid`：禁止并发运行，如果上一次运行尚未完成，则跳过下一次运行
>   - `Replace`：替换，取消当前正在运行的作业并用新作业替换它

创建 pc-cronjob.yaml，内容如下：

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: pc-cronjob
  namespace: dev
  labels:
    controller: cronjob
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    metadata:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: counter
            image: busybox:1.30
            command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 3;done"]
```

```bash
# 创建 cronjob
[root@k8s-master01 ~]# kubectl create -f pc-cronjob.yaml
cronjob.batch/pc-cronjob created

# 查看 cronjob
[root@k8s-master01 ~]# kubectl get cronjobs -n dev
NAME         SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
pc-cronjob   */1 * * * *   False     0        <none>          6s

# 查看 job
[root@k8s-master01 ~]# kubectl get jobs -n dev
NAME                    COMPLETIONS   DURATION   AGE
pc-cronjob-1592587800   1/1           28s        3m26s
pc-cronjob-1592587860   1/1           28s        2m26s
pc-cronjob-1592587920   1/1           28s        86s

# 查看 pod
[root@k8s-master01 ~]# kubectl get pods -n dev
pc-cronjob-1592587800-x4tsm   0/1     Completed   0          2m24s
pc-cronjob-1592587860-r5gv4   0/1     Completed   0          84s
pc-cronjob-1592587920-9dxxq   1/1     Running     0          24s

# 删除 cronjob
[root@k8s-master01 ~]# kubectl  delete -f pc-cronjob.yaml
cronjob.batch "pc-cronjob" deleted
```