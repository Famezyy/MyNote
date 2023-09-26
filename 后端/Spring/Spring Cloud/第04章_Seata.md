# 第04章_Seata

Seata 是一种开源的分布式事务解决方案，提供了 AT、TCC、SAGA 和 XA 事务模式，主要使用的是 AT 模式。阿里云上有商用的 GTS 服务。

官网：http://seata.io/zh-cn/index.html

## 1.常见的分布式事务解决方案

- Seata 分布式事务框架 - AT
- 消息队列 - TCC（Try-Confirm-Cancel）
- Saga
- XA

他们有一个共同点，都是“两阶段提交 - 2PC”，即把提交分为两个阶段：Prepare 和 Commit。

- 首先引入一个**协调者**管理各个参与者

- Prepare 阶段

  - 在事务开始时，协调者会向参与者发送 Prepare 请求

  - 各个参与者会检查各自的事务执行条件是否满足，并准备事务的业务操作
  - 如果协调者收到任一参与者的`ack:NO`应答后，会发送 rollback 请求

- Commit 阶段
  - 当协调者收到所有参与者的`ack:YES`应答后，会发送 Commit 请求
  - 参数者收到 Commit 请求后，会提交事务，并释放事务资源，成功后返回应答

### 1.1 2PC的问题

1. 同步阻塞，参与者在等待协调者的指令时，其实是在等待其他参与者的响应，在此过程中，参与者是无法进行其他操作的，也就是阻塞了其运行。倘若参与者与协调者之间网络异常导致参与者一直收不到协调者信息，那么会导致参与者一直阻塞下去。

2. 单点在 2PC 中，一切请求都来自协调者，所以协调者的地位是至关重要的，如果协调者宕机，那么就会使参与者一直阻塞并一直占用事务资源。

   如果协调者也是分布式，使用选主方式提供服务，那么在一个协调者挂掉后，可以选取另一个协调者继续后续的服务，可以解决单点问题。但是，新协调者无法知道上一个事务的全部状态信息（例如已等待 Prepare 响应的时长等），所以也无法顺利处理上一个事务。

3. 数据不一致 Commit 事务过程中 Commit/Rollback 请求可能因为协调者宕机或协调者与参与者网络问题丢失，那么就导致了部分参与者没有收到 Commit/Rollback 请求，而其他参与者则正常收到执行了 Commit/Rollback 操作，没有收到请求的参与者则继续阻塞。这时参与者之间的数据就不再一致了。

   当参与者执行 Commit/Rollback 后会向协调者发送 Ack，然而协调者不论是否收到所有的参与者的 Ack，该事务也不会再有其他补救措施了，协调者能做的也就是等待超时后像事务发起者返回一个“我不确定该事务是否成功”。

4. 环境可靠性依赖协调者 Prepare 请求发出后，等待响应，然而如果有参与者宕机或与协调者之间的网络中断，都会导致协调者无法收到所有参与者的响应，那么在 2PC 中，协调者会等待一定时间，然后超时后，会触发事务中断，在这个过程中，协调者和所有其他参与者都是出于阻塞的。这种机制对网络问题常见的现实环境来说太苛刻了。

### 1.2 AT模式（auto tansaction）

AT 模式是两阶段提交协议的演变，是一种无侵入的分布式事务解决方案。在 AT 模式下，用户只需关注业务 SQL，用户的 “业务 SQL” 作为一阶段，Seata 框架会自动生成事务的二阶段提交和回滚操作。

- 一阶段

  在一阶段，Seata 会拦截“业务 SQL”，首先解析 SQL 语义，找到“业务 SQL”要更新的业务数据，在业务数据被更新前，将其保存成“before image”，然后执行“业务 SQL”更新业务数据，在业务数据更新之后，再将其保存成“after image”，最后生成行锁。以上操作全部在一个数据库事务内完成，这样保证了一阶段操作的原子性。

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230501142302213.png" alt="image-20230501142302213" style="zoom:80%;" />

- 二阶段提交
  
  二阶段如果是提交的话，因为“业务 SQL”在一阶段已经提交至数据库，所以 Seata 框架只需将一阶段保存的快照数据和行锁删掉，完成数据清理即可。
  
  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230501142322820.png" alt="image-20230501142322820" style="zoom:80%;" />
  
- 二阶段回滚
  
  二阶段如果是回滚的话，Seata 就需要回滚一阶段已经执行的“业务 SQL”，还原业务数据。回滚方式便是用“before image”还原业务数据；但在还原前要首先要校验脏写，对比“数据库当前业务数据”和 “after image”，如果两份数据完全一致就说明没有脏写，可以还原业务数据，如果不一致就说明有脏写，出现脏写就需要转人工处理。
  
  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230501142411497.png" alt="image-20230501142411497" style="zoom:80%;" />

AT 模式的一阶段、二阶段提交和回滚均由 Seata 框架自动生成，用户只需编写“业务 SQL”，便能轻松接入分布式事务，AT 模式是一种对业务无任何侵入的分布式事务解决方案。

### 1.3 TCC模式

2PC 通常都是在跨库的 DB 层面，而 TCC 则在应用层面的处理，需要通过业务逻辑来实现。这种分布式事务的实现方式的优势在于，可以让应用自己定义数据操作的粒度，使得降低锁冲突、提高吞吐量成为可能。

TCC 模式需要用户根据自己的业务场景实现 Try、Confirm 和 Cancel 三个操作，侵入性较强；事务发起方在一阶段执行 Try 方式，在二阶段提交执行 Confirm 方法，二阶段回滚执行 Cancel 方法。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230501152236574.png" alt="image-20230501152236574" style="zoom: 80%;" />

TCC 三个方法描述：

- Try：资源的检测和预留
- Confirm：执行的业务操作提交，要求 Try 成功 Confirm 一定要能成功
- Cancel：预留资源释放

#### 1.TCC 的实践经验

##### 1.1 业务模型分2阶段设计

用户接入 TCC ，最重要的是考虑如何将自己的业务模型拆成两阶段来实现。 以“扣钱”场景为例，在接入 TCC 前，对 A 账户的扣钱，只需一条更新账户余额的 SQL 便能完成；但是在接入 TCC 之后，用户就需要考虑如何将原来一步就能完成的扣钱操作，拆成两阶段，实现成三个方法，并且保证一阶段 Try 成功的话 二阶段 Confirm 一定能成功。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230501173345046.png" alt="image-20230501173345046" style="zoom:80%;" />

##### 1.2 允许空回滚

Cancel 接口设计时需要允许空回滚。在 Try 接口因为丢包时没有收到，事务管理器会触发回滚，这时会触发 Cancel 接口，这时 Cancel 执行时发现没有对应的事务 xid 或主键时，需要返回回滚成功，让事务服务管理器认为已回滚，否则会不断重试，而 Cancel 又没有对应的业务数据可以进行回滚。

##### 1.3 防悬挂控制

悬挂的意思是：Cancel 比 Try 接口先执行，出现的原因是 Try 由于网络拥堵而超时，事务管理器生成回滚，触发 Cancel 接口，而最终又收到了 Try 接口调用，但是 Cancel 比 Try 先到。按照前面允许空回滚的逻辑，回滚会返回成功，事务管理器认为事务已回滚成功，则此时的 Try 接口不应该执行，否则会产生数据不一致，所以我们在 Cancel 空回滚返回成功之前先记录该条事务 xid 或业务主键，标识这条记录已经回滚过，Try 接口先检查这条事务 xid 或业务主键如果已经标记为回滚成功过，则不执行 Try 的业务操作。

##### 1.4 幂等控制

幂等性的意思是：对同一个系统，使用同样的条件，一次请求和重复的多次请求对系统资源的影响是一致的。因为网络抖动或拥堵可能会超时，事务管理器会对资源进行重试操作，所以很可能一个业务操作会被重复调用，为了不因为重复调用而多次占用资源，需要对服务设计时进行幂等控制，通常我们可以用事务 xid 或业务主键判重来控制。

### 1.4 Saga模式

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230501173746729.png" alt="image-20230501173746729" style="zoom:80%;" />

Saga 模式的实现，是长事务解决方案。Saga 是一种补偿协议，在 Saga 模式下，分布式事务内有多个参与者，每一个参与者都是一个冲正补偿服务，需要用户根据业务场景实现其正向操作和逆向回滚操作。

分布式事务执行过程中，依次执行各参与者的正向操作，如果所有正向操作均执行成功，那么分布式事务提交。如果任何一个正向操作执行失败，那么分布式事务会退回去执行前面各参与者的逆向回滚操作，回滚已提交的参与者，使分布式事务回到初始状态。Saga 正向服务与补偿服务也需要业务开发者实现。因此是业务入侵的。Saga 模式下分布式事务通常是由事件驱动的，各个参与者之间是异步执行的，Saga 模式是一种长事务解决方案。

Saga模式的优势是：

- 一阶段提交本地数据库事务，无锁，高性能
- 参与者可以采用事务驱动异步执行，高吞吐
- 补偿服务即正向服务的“反向”，易于理解，易于实现

缺点：Saga 模式由于一阶段已经提交本地数据库事务，且没有进行“预留”动作，所以不能保证隔离性

与 TCC 实践经验相同的是，Saga 模式中每个事务参与者的冲正、逆向操作，需要支持：

- 空补偿：逆向操作早于正向操作时
- 防悬挂控制：空补偿后要拒绝正向操作
- 幂等

### 1.5 XA模式

XA 是 X/Open DTP组织（X/Open DTP group）定义的两阶段提交协议，XA 被许多数据库（如 Oracle、DB2、SQL Server、MySQL）和中间件等工具（如 CICS 和 Tuxedo）本地支持 。 X/Open DTP 模型（1994）包括应用程序（AP）、事务管理器（TM）、资源管理器（RM）。XA 接口函数由数据库厂商提供。XA 规范的基础是两阶段提交协议 2PC。JTA(Java Transaction API) 是 Java 实现的 XA 规范的增强版接口。在 XA 模式下，需要有一个全局协调器，每一个数据库事务完成后，进行第一阶段预提交，并通知协调器，把结果给协调器。协调器等所有分支事务操作完成、都预提交后，进行第二步：协调器通知每个数据库进行逐个 commit/rollback。 其中，这个全局协调器就是 XA 模型中的 TM 角色，每个分支事务各自的数据库就是 RM。MySQL 提供的 XA 实现（https://dev.mysql.com/doc/refman/5.7/en/xa.html） XA 模式下的开源框架有 atomikos，其开发公司也有商业版本。

XA 模式缺点：事务粒度大。高并发下，系统可用性低，因此很少使用。

### 1.6 总结

- AT 模式是无侵入的分布式事务解决方案，适用于不希望对业务进行改造的场景

- TCC 模式是高性能分布式事务解决方案，适用于核心系统等对性能有很高要求的场景

- Saga 模式是长事务解决方案，适用于业务流程长且需要保证事务最终一致性的业务系统，Saga 模式一阶段就会提交本地事务，无锁，长流程情况下可以保证性能，多用于渠道层、集成层业务系统。事务参与者可能是其它公司的服务或者是遗留系统的服务，无法进行改造和提供 TCC 要求的接口，也可以使用 Saga 模式
- XA 模式是分布式强一致性的解决方案，但性能低而使用较少

## 2.Seata简介

Seata 是一款开源的分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务。Seata 将为用户提供了 AT、TCC、SAGA 和 XA 事务模式，为用户打造一站式的分布式解决方案。AT 模式是阿里首推的模式，阿里云上有商用版本的 GTS（Global Transaction Service 全局事务服务）。

### 2.1 Seata的三大角色

在 Seata 的架构中，一共有三个角色：

- TC (Transaction Coordinator) - 事务协调者：维护全局和分支事务的状态，驱动全局事务提交或回滚
- TM (Transaction Manager) - 事务管理器：定义全局事务的范围，开始全局事务、提交或回滚全局事务
- RM (Resource Manager) - 资源管理器：管理分支事务处理的资源，与 TC 交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚

其中，TC 为单独部署的 Server 服务端，TM 和 RM 为嵌入到应用中的 Client 客户端。

在 Seata 中，一个分布式事务的生命周期如下：

1. TM 请求 TC 开启一个全局事务。TC 会生成一个 XID 作为该全局事务的编号。XID，会在微服务的调用链路中传播，保证将多个微服务的子事务关联在一起。当一进入事务方法中就会生成 XID， global_table 就是存储的全局事务信息
2. RM 请求 TC 将本地事务注册为全局事务的分支事务，通过全局事务的 XID 进行关联。当运行数据库操作方法，branch_table 存储事务参与者。
3. TM 请求 TC 告诉 XID 对应的全局事务是进行提交还是回滚。
4. TC 驱动 RM 们将 XID 对应的自己的本地事务进行提交还是回滚。

### 2.2 设计思路

AT 模式的核心是对业务无侵入，是一种改进后的两阶段提交，其设计思路如图

#### 第一阶段

业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。核心在于对业务 sql 进行解析，转换成 undolog，并同时入库。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230501194853593.png" alt="image-20230501194853593" style="zoom:67%;" />

#### 第二阶段

分布式事务操作成功，则 TC 通知 RM 异步删除 undolog。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230501194923026.png" alt="image-20230501194923026" style="zoom:67%;" />

分布式事务操作失败，TM 向 TC 发送回滚请求，RM 收到协调器 TC 发来的回滚请求，通过 XID 和 Branch ID 找到相应的回滚日志记录，通过回滚记录生成反向的更新 SQL 并执行，以完成分支的回滚。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230501194949168.png" alt="image-20230501194949168" style="zoom:67%;" />

### 2.3 整体执行流程

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230501195022873.png" alt="image-20230501195022873" style="zoom:67%;" />

### 2.4 设计亮点

相比与其它分布式事务框架，Seata 架构的亮点主要有几个: 

- 应用层基于 SQL 解析实现了自动补偿，从而最大程度的降低业务侵入性
- 将分布式事务中 TC（事务协调者）独立部署，负责事务的注册、回滚
- 通过全局锁实现了写隔离与读隔离

### 2.5 存在的问题

- **性能损耗**

  一条 Update 的 SQL，则需要全局事务 xid 获取（与 TC 通讯）、before image（解析 SQL，查询一次数据库）、after image（查询一次数据库）、insert undo log（写一次数据库）、before commit（与 TC 通讯，判断锁冲突），这些操作都需要一次远程通讯 RPC，而且是同步的。另外 undo log 写入时 blob 字段的插入性能也是不高的。每条写 SQL 都会增加这么多开销，粗略估计会增加 5 倍响应时间。

- **性价比**

  为了进行自动补偿，需要对所有交易生成前后镜像并持久化。

- **全局锁**

  - 热点数据

    相比 XA，Seata 虽然在一阶段成功后会释放数据库锁，但一阶段在 commit 前全局锁的判定也拉长了对数据锁的占有时间，这个开销比 XA 的 prepare 低多少需要根据实际业务场景进行测试。全局锁的引入实现了隔离性，但带来的问题就是阻塞，降低并发性，尤其是热点数据，这个问题会更加严重。

  - 回滚锁释放时间

    Seata 在回滚时，需要先删除各节点的 undo log，然后才能释放 TC 内存中的锁，所以如果第二阶段是回滚，释放锁的时间会更长。

  - 死锁问题

    Seata 的引入全局锁会额外增加死锁的风险，但如果出现死锁会不断进行重试，最后靠等待全局锁超时，这种方式并不优雅，也延长了对数据库锁的占有时间。

## 3.快速开始

### 3.1 Seata Server（TC）环境搭建

官网：https://seata.io/zh-cn/docs/ops/deploy-guide-beginner.html

Server 端存储模式（store.mode）现有 file、db、redis 三种（后续将引入 raft、mongodb），file 模式无需改动，直接启动即可，下面专门讲下 db 和 redis 启动步骤。

- file 模式为单机模式，全局事务会话信息内存中读写并持久化本地文件 root.data，性能较高（默认）
- db 模式为高可用模式，全局事务会话信息通过 db 共享，相应性能差些
- redis 模式 Seata-Server 1.3 及以上版本支持，性能较高，存在事务信息丢失风险，请提前配置合适当前场景的 redis 持久化配置

#### 1.JAR包启动

##### 步骤一：启动包

[点击下载](https://github.com/seata/seata/releases)

##### 步骤二：建表（仅 db）

[建表语句](https://github.com/seata/seata/tree/master/script/server/db)

全局事务会话信息由3块内容构成，全局事务-->分支事务-->全局锁，对应表 global_table、branch_table、lock_table

##### 步骤三：修改 store.mode

- 启动包：seata-->conf-->application.yml，修改 store.mode="db 或者 redis"

- 源码：根目录-->seata-server-->resources-->application.yml，修改 store.mode="db 或者 redis"

1.5.0 以下版本:

- 启动包：seata-->conf-->file.conf，修改 store.mode="db 或者 redis"

- 源码：根目录-->seata-server-->resources-->file.conf，修改 store.mode="db 或者 redis"

同时还要修改数据库 URL 和用户名密码。

##### 步骤四：修改数据库连接|redis 属性配置

- 启动包：seata-->conf-->application.example.yml 中附带额外配置，将其 db|redis 相关配置复制至 application.yml，进行修改 store.db 或 store.redis 相关属性。

- 源码：根目录-->seata-server-->resources-->application.example.yml 中附带额外配置，将其 db|redis 相关配置复制至 application.yml，进行修改 store.db 或 store.redis 相关属性。

1.5.0 以下版本:

- 启动包：seata-->conf-->file.conf，修改 store.db 或 store.redis 相关属性。
- 源码：根目录-->seata-server-->resources-->file.conf，修改 store.db 或 store.redis 相关属性。

##### 步骤五：启动

- 源码启动: 执行`ServerApplication.java`的`main`方法
- 命令启动: [seata-server.sh](http://seata-server.sh/) -h 127.0.0.1 -p 8091 -m db

1.5.0 以下版本

- 源码启动: 执行`Server.java`的`main`方法
- 命令启动: [seata-server.sh](http://seata-server.sh/) -h 127.0.0.1 -p 8091 -m db -n 1 -e test

```bash
-h: 注册到注册中心的 ip
-p: Server rpc 监听端口
-m: 全局事务会话信息存储模式，file、db、redis，优先读取启动参数 (Seata-Server 1.3 及以上版本支持 redis)
-n: Server node，多个 Server 时，需区分各自节点，用于生成不同区间的 transactionId，以免冲突
-e: 多环境配置参考 http://seata.io/en-us/docs/ops/multi-configuration-isolation.html
```

注: 堆内存建议分配 2G，堆外内存 1G

#### 2.docker启动

[docker部署](https://seata.io/zh-cn/docs/ops/deploy-by-docker.html)

##### 2.1 启动seata-server实例

```bash
$ docker run --name seata-server -p 8091:8091 -p 7091:7091 seataio/seata-server
```

##### 2.2 指定seata-server IP和端口启动

```bash
$ docker run --name seata-server \
        -p 8091:8091 \
        -p 7091:7091 \
        -e SEATA_IP=192.168.1.1 \
        -e SEATA_PORT=8091 \
        seataio/seata-server
```

##### 2.3 容器命令行及查看日志

```bash
$ docker exec -it seata-server sh
$ docker logs -f seata-server
```

##### 2.4 使用自定义配置文件

自定义配置文件需要通过挂载文件的方式实现，将宿主机上的`application.yml`挂载到容器中相应的目录。

首先启动一个用户将 resources 目录文件拷出的临时容器

```docker
docker run -d -p 8091:8091 -p 7091:7091 --name seata-serve seataio/seata-server:latest
docker cp seata-serve:/seata-server/resources /youyi/seata/config
```

拷出后可以，可以选择修改 application.yml 再 cp 进容器，或者 rm 临时容器，然后重新创建，并做好映射路径设置。

修改配置文件

```yaml
store:
    # support: file 、 db 、 redis
    mode: db
    db:
      datasource: druid
      db-type: mysql
      driver-class-name: com.mysql.jdbc.Driver
      url: jdbc:mysql://192.168.11.100:3307/seata?rewriteBatchedStatements=true
      user: seata
      password: seata
      min-conn: 5
      max-conn: 100
      global-table: global_table
      branch-table: branch_table
      lock-table: lock_table
      distributed-lock-table: distributed_lock
      query-limit: 100
      max-wait: 5000
```

##### 2.5 指定application.yml

```bash
$ docker run --name seata-server \
        -p 8091:8091 \
        -p 7091:7091 \
        -v /youyi/seata/config:/seata-server/resources  \
        seataio/seata-server:1.5.0
```

其中`-e`用于配置环境变量，`-v`用于挂载宿主机的目录，如果是以 file 存储模式运行，请加上`-v /User/seata/sessionStore :/seata-server/sessionStore`将 file 的数据文件映射到宿主机，以防数据丢失。

接下来你可以看到宿主机对应目录下已经有了，logback-spring.xml，application.example.yml，application.yml。

##### 2.6 环境变量

seata-server 支持以下环境变量：

- **SEATA_IP**

  可选, 指定 seata-server 启动的 IP, 该 IP 用于向注册中心注册时使用, 如 eureka 等。

- **SEATA_PORT**

  可选, 指定 seata-server 启动的端口, 默认为`8091`。

- **STORE_MODE**

  可选, 指定 seata-server 的事务日志存储方式, 支持`db`、`file`、redis（Seata-Server 1.3 及以上版本支持），默认是`file`。

- **SERVER_NODE**

  可选, 用于指定 seata-server 节点 ID, 如 `1`,`2`,`3`..., 默认为`根据ip生成`。

- **SEATA_ENV**

  可选, 指定 seata-server 运行环境, 如`dev`，`test`等, 服务启动时会使用`registry-dev.conf`这样的配置。

- **SEATA_CONFIG_NAME**

  可选, 指定配置文件位置, 如`file:/root/registry`，将会加载`/root/registry.conf`作为配置文件，如果需要同时指定`file.conf`文件，需要将`registry.conf`的`config.file.name`的值改为类似`file:/root/file.conf`。

##### 2.7 Docker compose启动

`docker-compose.yaml`示例

```yaml
version: "3"
services:
  seata-server:
    image: seataio/seata-server
    container_name: seata-server
    ports:
      - "8091:8091"
      - "7091:7091"
    volumes:
      - /youyi/seata/config/resources:/seata-server/resources
    environment:
      - SEATA_PORT=8091
      - STORE_MODE=db
    restart: always
    depends_on:
      mysql:
        condition: service_healthy

  mysql:
    build:
      context: ./
      dockerfile: mysql/Dockerfile
    image: seata-mysql
    container_name: mysql-seata
    ports:
      - "3307:3306"
    volumes:
      - /youyi/mysql-seata/log:/var/log/mysql
      - /youyi/mysql-seata/data:/var/lib/mysql
      - /youyi/mysql-seata/conf:/etc/mysql/conf.d
    env_file:
      - ./mysql/mysql.env
    restart: always
    healthcheck:
      test: [ "CMD", "mysqladmin" ,"ping", "-h", "localhost" ]
      interval: 5s
      timeout: 10s
      retries: 10
```

**`./mysql/Dockerfile`**

```dockerfile
FROM mysql:5.7.40
ADD https://raw.githubusercontent.com/seata/seata/master/script/server/db/mysql.sql /docker-entrypoint-initdb.d/seata-mysql.sql
RUN chown -R mysql:mysql /docker-entrypoint-initdb.d/seata-mysql.sql
EXPOSE 3307
CMD ["mysqld", "--character-set-server=utf8mb4", "--collation-server=utf8mb4_unicode_ci"]
```

**`./mysql/mysql.env`**

```properties
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=seata
MYSQL_USER=seata
MYSQL_PASSWORD=seata
```

**`/youyi/seata/config/resources`**

将修改好的自定义配置文件（2.4）放在这个文件夹下。

启动

```bash
docker compose up -d
```

#### 3.配置Nacos

- 在 application.yml 中修改 Nacos 注册中心和配置中心地址

  ```yaml
  seata:
    config:
      type: nacos
      nacos:
        server-addr: 192.168.11.100:8848
        group : SEATA_GROUP
        namespace: public
        username: nacos
        password: nacos
        data-id: seataServer.properties
    registry:
      type: nacos
      nacos:
        application: seata-server
        server-addr: 192.168.11.100:8848
        group : SEATA_GROUP
        namespace: public
        username: nacos
        password: nacos
  ```

- 上传配置到 Nacos，参考：http://seata.io/zh-cn/docs/user/configuration/nacos.html

  在 Nacos 新建配置，此处 dataId 为 seataServer.properties，配置内容参考 https://github.com/seata/seata/tree/develop/script/config-center 的 config.txt 并按需修改保存

  ```properties
  store.mode=db
  
  #删除store.file相关配置
  
  store.db.url=jdbc:mysql://192.168.11.100:3307/seata?rewriteBatchedStatements=true
  store.db.user=seata
  store.db.password=seata
  ```

- 启动 seata

  