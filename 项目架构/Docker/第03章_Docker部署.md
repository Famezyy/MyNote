# 第03章_Docker部署

## 1.常用软件安装

https://hub.docker.com/_/tomcat

总体步骤：搜索镜像 - 拉取镜像 - 查看镜像 - 启动镜像（服务端口映射） - 停止容器 - 移除容器

### 5.1 Tomcat

```bash
$ docker search tomcat
$ docker pull tomcat
$ docker images
$ docker run -d -p 8080:8080 tomcat
```

新版中 tomcat 首页不能正常访问，需要将`webapps.dist`下的文件放入`webapps`中：

```bash
$ cp -r webapps.dist/* ./webapps
```

使用以下版本可以直接打开首页

```bash
$ docker pull billygoo/tomcat8-jdk8
$ docker run -d -p 8080:8080 --name mytomcat8 billygoo/tomcat8-jdk8
```

### 5.2 Mysql

https://hub.docker.com/_/mysql

```bash
$ docker pull mysql:5.7
$ docker run -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7
```

默认启动时使用的是`latin1`字符集，不支持中文，需要创建配置文件。

因此推荐在启动时使用容器数据卷映像配置文件：

```bash
$ docker run -d -p 3306:3306 --privileged=true -v /youyi/mysql/log:/var/log/mysql -v /youyi/mysql/data:/var/lib/mysql -v /youyi/mysql/conf:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456  --name mysql_youyi mysql:5.7
```

在本机的`/youyi/mysql/conf`下新建`my.cnf`：

```bash
[client]
default_character_set=utf8
[mysqld]
collation_server = utf8_general_ci
character_set_server = utf8
```

重新启动 mysql 容器实例再重新进入并查看字符编码：

```bash
$ docker restart mysql
$ docker exec -it mysql bash
```

```mysql
SHOW VARIABLES LIKE 'character%';
```

### 5.3 Redis

docker 中的 redis 没有配置文件，先在本机创建好配置文件

```bash
$ wget https://download.redis.io/redis-stable.tar.gz
$ tar -xzvf redis-stable.tar.gz
$ cp redis-stable/redis.conf /app/redis/redis.conf
```

新建一个`redis-node.conf`配置文件：

```bash
include /etc/redis/redis.conf
# 开启密码验证（可选）
requirepass 123456
# 允许 redis 外地连接（必须）
bind 0.0.0.0
# 开启 redis AOF 持久化(可选)
appendonly yes
# 关闭保护模式以允许其他主机访问
protected-mode no
```

挂载目录运行

```bash
$ docker pull redis:6.0.8
$ docker run -d -p 6379:6379 --privileged=true \
-v /app/redis/redis.conf:/etc/redis/redis.conf \
-v /app/redis/redis-node.conf:/etc/redis/redis-node.conf \
-v /app/redis/data:/data \
--name myr3 \
redis \
redis-server /etc/redis/redis-node.conf
```

redis 容器中默认保存了 RDB 快照文件，放在容器的 data 目录，重启 redis 时会重新加载这个快照文件，因此数据不会丢失。

## 2.Mysql主从搭建

### 2.1 新建主服务器容器

```bash
$ docker run -d -p 3306:3306 --privileged=true -v /youyi/mysql_master/log:/var/log/mysql -v /youyi/mysql_master/data:/var/lib/mysql -v /youyi/mysql_master/conf:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456  --name mysql_master mysql:5.7
```

- 进入`/youyi/mysql-master/conf`目录下新建`my.cnf`

  ```bash
  [mysqld]
  #[必须] 设置server_id，同一局域网中需要唯一
  server_id=1
  #[必须] 开启二进制日志功能，指定路径
  log-bin=mall-mysql-bin
  
  #[可选] 0（默认）表示读写（主机），1表示只读（从机）
  read-only=0
  
  #[可选] 设置需要复制的数据库，默认全部记录。比如：binlog-do-db=youyi_master_slave
  binlog-do-db=需要复制的主数据库名字
  #[可选] 指定不需要同步的数据库名称
  binlog-ignore-db=mysql  
  
  #[可选] 二进制日志过期清理时间，默认值为 0，表示不自动清理
  expire_logs_days=7  
  #[可选] 设置日志文件保留的时长，单位是秒
  binlog_expire_logs_seconds=6000
  
  #[可选] 设置二进制日志使用内存大小（事务）
  binlog_cache_size=1M
  #[可选] 控制单个二进制日志大小。此参数的最大和默认值是 1GB
  max_binlog_size=200M
  
  #[建议] 设置使用的二进制日志格式（mixed,statement,row）
  binlog_format=mixed  
  #[可选] 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断
  ## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
  slave_skip_errors=1062
  
  collation_server=utf8_general_ci
  character_set_server=utf8
  [client]
  default_character_set=utf8
  ```

- 修改完配置后重启 master 实例

  ```bash
  $ docker restart mysql_master
  ```

- 进入 mysql-master 容器

  ```bash
  $ docker exec -it mysql-master /bin/bash
  $ mysql -uroot -proot
  ```

- master 容器实例内创建数据同步用户

  ```bash
  $ CREATE USER 'slave'@'%' IDENTIFIED BY '123456';
  $ GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';
  ```

  > **注意：**
  >
  > 如果使用的是`mysql8`，需要如下的方式建立账户，并授权 slave
  >
  > ```bash
  > CREATE USER 'slave1'@'%' IDENTIFIED BY '123456';
  > 
  > GRANT REPLICATION SLAVE ON *.* TO 'slave1'@'%';
  > GRANT REPLICATION CLIENT ON *.* TO 'slave1'@'%';
  > 
  > #此语句必须执行，否则出错
  > ALTER USER 'slave1'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
  > 
  > flush privileges;
  > ```
  >

- 查询 Master 的状态，并记录下 File 和 Position 的值

  ```mysql
  mysql> show master status;
  ```

### 2.2 新建从服务器容器

```bash
$ docker run -d -p 3307:3306 --privileged=true -v /youyi/mysql_slave/log:/var/log/mysql -v /youyi/mysql_slave/data:/var/lib/mysql -v /youyi/mysql_slave/conf:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456  --name mysql_slave mysql:5.7
```

- 进入`/youyi/mysql_slave/conf`目录下新建`my.cnf`

  ```bash
  [mysqld]
  # 设置server_id，同一局域网中需要唯一
  server_id=2
  # relay_log 配置中继日志文件名
  relay_log=mall-mysql-relay-bin
  # slave设置为只读（具有 super 权限的用户除外）
  read_only=1
  
  # 开启二进制日志功能，以备 Slave 作为其它数据库实例的 Master 时使用
  log-bin=mall-mysql-slave1-bin  
  ## log_slave_updates 表示 slave 将复制事件写进自己的二进制日志
  log_slave_updates=1  
  ## 设置使用的二进制日志格式（mixed,statement,row）
  binlog_format=mixed  
  ## 跳过主从复制中遇到的所有错误或指定类型的错误，避免 slave 端复制中断
  ## 如：1062 错误是指一些主键重复，1032错误是因为主从数据库数据不一致
  slave_skip_errors=1062
  
  #[可选] 设置需要复制的数据库，默认全部记录，比如：binlog-do-db=youyi_master_slave
  binlog-do-db=需要复制的主数据库名字
  #[可选] 指定不需要同步的数据库名称
  binlog-ignore-db=mysql
  
  #[可选] 设置二进制日志使用内存大小（事务）
  binlog_cache_size=1M
  #[可选] 控制单个二进制日志大小。此参数的最大和默认值是 1GB
  max_binlog_size=200M
  
  #[可选] 二进制日志过期清理时间，默认值为 0，表示不自动清理
  expire_logs_days=7  
  #[可选] 设置日志文件保留的时长，单位是秒
  binlog_expire_logs_seconds=6000
  
  collation_server=utf8_general_ci
  character_set_server=utf8
  [client]
  default_character_set=utf8
  ```

- 修改完配置后重启 slave 实例

  ```bash
  $ docker restart mysql-slave
  ```

- 进入 mysql-slave 容器

  ```bash
  $ docker exec -it mysql-slave /bin/bash
  $ mysql -uroot -proot
  ```

- 在从数据库中配置主从复制

  ```mysql
  change master to master_host='宿主机ip', master_user='slave', master_password='123456', master_port=3307, master_log_file='mall-mysql-bin.000001', master_log_pos=617, master_connect_retry=30;
  ```

  ```mysql
  CHANGE MASTER TO
  MASTER_HOST='主机的IP地址',
  MASTER_PORT='主机的端口',
  MASTER_USER='主机用户名',
  MASTER_PASSWORD='主机用户名的密码',
  MASTER_LOG_FILE='mysql-bin.具体数字',
  MASTER_LOG_POS=主机偏移量,
  MASTER_CONNECT_RETRY=连接失败重试的时间间隔，单位为秒;
  ```

- 在从数据库中查看主从同步状态

  ```mysql
  show slave status \G
  ```

- 在从数据库中开启主从同步

  ```mysql
  start slave;
  ```

- 查看从数据库状态发现已经同步

  ```bash
  Slave_IO_Running: Yes
  Slave_SQL_Running: Yes
  ```

> **注意**
>
> 如果报错，可以执行`reset slave`删除 SLAVE 数据库的 relaylog 日志文件，并重新启用新的 relaylog 文件，然后重新执行`CHANGE MASTER TO …`语句即可。
>
> 停止主从时执行`stop slave`。

## 3.Redis集群搭建

### 3.1 哈希槽分区

哈希槽实质就是一个数组，数组`[0,2^14 -1]`形成 hash slot 空间。

Redis 集群模式为了解决单机 Redis 容量有限的问题，将数据按一定的规则分配到多台机器，内存/QPS不受限于单机，可受益于分布式集群高扩展性。Redis 集群模式是一种服务器 Sharding 技术（分片和路由都是在服务端实现），**采用多主多从，每一个分区都是由一个Redis主机和多个从机组成，片区和片区之间是相互平行的**。Redis Cluster 集群采用了 P2P 的模式，完全去中心化。

一个集群只能有 16384 个槽，编号 0-16383（0-2^14-1）。这些槽会分配给集群中的所有主节点，分配策略没有要求。可以指定哪些编号的槽分配给哪个主节点。集群会记录节点和槽的对应关系。解决了节点和槽的关系后，接下来就需要对 key 求哈希值，然后对 16384 取余，余数是几 key 就落入对应的槽里（`slot = CRC16(key) % 16384`）。以槽为单位移动数据，因为槽的数目是固定的，处理起来比较容易，这样数据移动问题就解决了。

**哈希槽计算**

Redis 集群中内置了 16384 个哈希槽，redis 会根据节点数量大致均等的将哈希槽映射到不同的节点。当需要在 Redis 集群中放置一个 key-value 时，redis 先对 key 使用 crc16 算法算出一个结果，然后把结果对 16384 求余数，这样每个 key 都会对应一个编号在 0-16383 之间的哈希槽，也就是映射到某个节点上。如下代码，key 之 A 、B 在 Node2， key 之 C 落在 Node3 上：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230121190827447-b9a8299d17a335489c92489f6abaa9ce-749934.png" alt="image-20230121190827447" style="zoom:67%;" />

### 3.2 哨兵配置

**主从配置**

- 新建主机配置文件`redis_master.conf`

  ```bash
  include /etc/redis/redis.conf
  port 6379
  bind 0.0.0.0
  protected-mode no
  appendonly yes
  requirepass 123456
  masterauth 123456
  ```

- 启动主机

  ```bash
  $ docker run -d --net=host --privileged=true \
  -v /tmp/redis.conf:/etc/redis/redis.conf \
  -v /tmp/redis_master.conf:/etc/redis/redis_master.conf \
  -v /tmp/redis_master/data:/data \
  --name redis_master \
  redis \
  redis-server /etc/redis/redis_master.conf
  ```

- 新建从机配置文件`redis_slave.conf`

  ```bash
  include /etc/redis/redis.conf
  port 6380
  bind 0.0.0.0
  protected-mode no
  appendonly yes
  requirepass 123456
  masterauth 123456
  slaveof 192.168.11.101 6379
  ```

- 启动从机

  ```bash
  $ docker run -d --net=host --privileged=true \
  -v /tmp/redis.conf:/etc/redis/redis.conf \
  -v /tmp/redis_slave/data:/data \
  --name redis_slave \
  redis \
  redis-server /etc/redis/redis_slave.conf
  ```

- 在原哨兵配置文件`sentinel.conf`中修改

  ```bash
  sentinel monitor mymaster 192.168.11.101 6379 1
  sentinel auth-pass mymaster 123456
  logfile "/var/log/sentinel.log"
  daemonize yes
  ```

- 启动哨兵容器

  ```bash
  $ docker run -it --net=host -v /tmp/sentinel.conf:/etc/redis/sentinel.conf --name redis_sentinel redis bash
  $ redis-sentinel /etc/redis/sentinel.conf
  ```

**测试**

- 进入主机

  ```bash
  $ docker exec -it redis_master bash
  $ redis-cli -a 123456
  ```

- 插入值后进入从机

  ```bash
  $ redis-cli -p 6380
  ```

- 查询该值发现能查到

- 主机下线后从机自动变为主机

**SpringBoot 配置**

```properties
# 哨兵监听的 redis server 名称
spring.redis.sentinel.master=mymaster
# 哨兵的端口
spring.redis.sentinel.nodes=192.168.11.101:26379
spring.redis.password=123456
```

**两台服务器1主1从自动脚本**

- 启动主节点

  - 将原始`redis.conf`配置文件放在脚本同一目录

  - 执行主服务器脚本

    ```bash
    $ chmod +x redis_master.sh
    $ ./redis-master.sh 192.168.11.101:6379 # ip修改为当前服务器地址
    ```

- 启动从节点

  - 将原始`redis.conf`配置文件放在脚本同一目录

  - 执行从服务器脚本

    ```bash
    $ chmod +x redis_slave.sh
    $ ./redis-master.sh 192.168.11.101:6379 192.168.11.102:6379  #ip1 为主节点服务器地址 #ip2 为当前服务器地址
    ```

- 验证节点是否正常

  ```bash
  #进入任一哨兵容器
  docker exec -it redis_sentinel_26679 bash
  #登录redis（替换ip）
  redis-cli -h 127.0.0.1 -p 26679
  #查看主节点
  SENTINEL masters
  #查看从节点
  SENTINEL slaves mymaster
  #查看集群信息
  info
  ```

- 主服务器脚本

  ```bash
  #!/bin/bash
  
  # 接受外部参数
  IP_ADDRESS=$1
  IFS=: read -r local_ip master_port <<< "$IP_ADDRESS"
  echo "====1.接受外部参数 local_ip:$local_ip local_port:$master_port===="
  
  # 哨兵节点端口
  sentinel_port="26379"
  
  rm -rf /local/redis_master
  echo "====2.开始安装主节点===="
  sudo mkdir -p /local/redis_master/redis_$master_port/log
  sudo mkdir -p /local/redis_master/redis_$master_port/data
  # 授权
  chmod 777 /local/redis_master/redis_$master_port/log
  chmod 777 /local/redis_master/redis_$master_port/data
  
  # 将同文件夹下的 redis.conf 复制到相应目录
  echo "复制原始 redis.conf，将自动修改配置"
  REDIS_CONF=./redis.conf
  if test ! -f "$REDIS_CONF"; then
  	echo "$REDIS_CONF not exist"
  	exit
  fi
  cp ./redis.conf /local/redis_master/redis_$master_port/redis.conf
  
  # 创建主节点配置文件
  cat > /local/redis_master/redis_$master_port/redis_$master_port.conf << EOF
  include /etc/redis/redis.conf
  port $master_port
  pidfile /var/run/redis_master_$master_port.pid
  dbfilename dump_$master_port.rdb
  logfile "/var/log/redis_master_$master_port.log"
  bind 0.0.0.0
  protected-mode no
  appendonly yes
  save 900 1
  save 300 10
  save 60 10000
  requirepass 123456
  masterauth 123456
  EOF
  
  echo "====3.开始启动节点容器===="
  sudo docker run \
  -d \
  --network host \
  --restart=always \
  --name redis_master_$master_port \
  --privileged=true \
  -v /local/redis_master/redis_$master_port/redis_$master_port.conf:/etc/redis/redis_$master_port.conf \
  -v /local/redis_master/redis_$master_port/redis.conf:/etc/redis/redis.conf \
  -v /local/redis_master/redis_$master_port/data:/data \
  -v /local/redis_master/redis_$master_port/log:/var/log \
  -v /etc/localtime:/etc/localtime:ro \
  redis \
  redis-server /etc/redis/redis_$master_port.conf
  
  echo "====4.开始安装哨兵节点===="
  sudo mkdir -p /local/redis_master/sentinel_$sentinel_port/log
  sudo mkdir -p /local/redis_master/sentinel_$sentinel_port/data
  # 授权
  chmod 777 /local/redis_master/sentinel_$sentinel_port/log
  chmod 777 /local/redis_master/sentinel_$sentinel_port/data
  
  #创建哨兵配置文件
  cat > /local/redis_master/sentinel_$sentinel_port/sentinel.conf << EOF
  pidfile /var/run/master_sentinel_$sentinel_port.pid
  port $sentinel_port
  protected-mode no
  daemonize yes
  logfile "/var/log/master_sentinel_$sentinel_port.log"
  sentinel announce-ip $local_ip
  sentinel announce-port $sentinel_port
  # 总哨兵数量 / 2 + 1
  sentinel monitor mymaster ${local_ip} ${master_port} 2
  sentinel auth-pass mymaster 123456
  sentinel down-after-milliseconds mymaster 5000
  EOF
  
  # 启动哨兵容器
  echo "====5.启动哨兵容器===="
  sudo docker run -it \
  --name master_sentinel_$sentinel_port \
  --net=host \
  --restart=always \
  -v /local/redis_master/sentinel_$sentinel_port/sentinel.conf:/etc/redis/sentinel.conf \
  -v /local/redis_master/sentinel_$sentinel_port/log:/var/log \
  -v /etc/localtime:/etc/localtime:ro \
  -d redis /bin/bash
  
  # 启动哨兵
  echo "====5.启动哨兵===="
  sudo docker exec -it master_sentinel_$sentinel_port /bin/bash -c 'redis-sentinel /etc/redis/sentinel.conf'
  ```

- 从服务器脚本

  ```bash
  #!/bin/bash
  
  # 接收外部参数
  MASTER_ADDRESS=$1
  SLAVE_ADDRESS=$2
  IFS=: read -r master_ip master_port <<< "$MASTER_ADDRESS"
  IFS=: read -r slave_ip slave_port <<< "$SLAVE_ADDRESS"
  echo "====1.接受外部参数 master_ip:$master_ip master_port:$master_port===="
  echo "====1.接收外部参数 slave_ip:$slave_ip slave_port:$slave_port ===="
  
  # 哨兵节点端口
  # 修改哨兵数量时记得修改哨兵配置文件的 sentinel monitor mymaster
  sentinel_ports=("26379" "26380")
  
  rm -rf /local/redis_slave
  echo "====2.开始安装从节点===="
  sudo mkdir -p /local/redis_slave/redis_$slave_port/log
  sudo mkdir -p /local/redis_slave/redis_$slave_port/data
  # 授权
  chmod 777 /local/redis_slave/redis_$slave_port/log
  chmod 777 /local/redis_slave/redis_$slave_port/data
  
  # 将同文件夹下的 redis.conf 复制到相应目录
  echo "复制原始 redis.conf，将自动修改配置"
  REDIS_CONF=./redis.conf
  if test ! -f "$REDIS_CONF"; then
  	echo "$REDIS_CONF not exist"
  	exit
  fi
  cp ./redis.conf /local/redis_slave/redis_$slave_port/redis.conf
  
  # 创建从节点配置文件
  cat > /local/redis_slave/redis_$slave_port/redis_$slave_port.conf << EOF
  include /etc/redis/redis.conf
  port $slave_port
  pidfile /var/run/redis_slave_$slave_port.pid
  dbfilename dump_$slave_port.rdb
  logfile "/var/log/redis_slave_$slave_port.log"
  bind 0.0.0.0
  protected-mode no
  appendonly yes
  save 900 1
  save 300 10
  save 60 10000
  requirepass 123456
  masterauth 123456
  slaveof $master_ip $master_port
  EOF
  
  echo "====3.开始启动节点容器===="
  sudo docker run \
  -d \
  --network host \
  --restart=always \
  --name redis_slave_$slave_port \
  --privileged=true \
  -v /local/redis_slave/redis_$slave_port/redis_$slave_port.conf:/etc/redis/redis_$slave_port.conf \
  -v /local/redis_slave/redis_$slave_port/redis.conf:/etc/redis/redis.conf \
  -v /local/redis_slave/redis_$slave_port/data:/data \
  -v /local/redis_slave/redis_$slave_port/log:/var/log \
  -v /etc/localtime:/etc/localtime:ro \
  redis \
  redis-server /etc/redis/redis_$slave_port.conf
  
  echo "====4.开始安装哨兵节点===="
  for port in ${sentinel_ports[@]}
  do
  #创建宿主机挂在目录
  sudo mkdir -p /local/redis_slave/sentinel_$port/log
  sudo mkdir -p /local/redis_slave/sentinel_$port/data
  #授权
  chmod 777 /local/redis_slave/sentinel_$port/log
  chmod 777 /local/redis_slave/sentinel_$port/data
  
  # 创建哨兵配置文件
  cat > /local/redis_slave/sentinel_$port/sentinel.conf << EOF
  pidfile /var/run/slave_sentinel_$port.pid
  port $port
  protected-mode no
  daemonize yes
  logfile "/var/log/slave_sentinel_$port.log"
  sentinel announce-ip $slave_ip
  sentinel announce-port $port
  sentinel monitor mymaster ${master_ip} ${master_port} 2
  sentinel auth-pass mymaster 123456
  sentinel down-after-milliseconds mymaster 5000
  EOF
  done
  
  # 启动哨兵容器
  for port in ${sentinel_ports[@]}
  do
      sudo docker run -it \
      --name slave_sentinel_$port \
      --net=host \
      --restart=always \
      -v /local/redis_slave/sentinel_$port/sentinel.conf:/etc/redis/sentinel.conf \
      -v /local/redis_slave/sentinel_$port/log:/var/log \
      -v /etc/localtime:/etc/localtime:ro \
      -d redis /bin/bash
  done
  
  # 启动哨兵
  echo "====5.启动哨兵===="
  for port in ${sentinel_ports[@]}
      do
      sudo docker exec -it slave_sentinel_$port /bin/bash -c 'redis-sentinel /etc/redis/sentinel.conf'
  done
  ```

### 3.3 集群配置

配置 3 主 3 从集群：Master1 - Slave1；Master2 - Slave2；Master3 - Slave3

**分别在二台服务器上执行以下操作**

- 开放相应端口

  例：

  ```bash
  # 查看所有端口
  $ firewall-cmd --list-ports
  # 添加 6391 端口
  $ firewall-cmd --zone=public --add-port=6391/tcp --permanent
  # 刷新防火墙
  $ firewall-cmd --reload
  ```

- 新建配置文件

  ```bash
  $ for port in $(seq 6381 6383); \
  do \
  mkdir -p /tmp/apps/redis_node_${port}/{conf,data}
  cat > /tmp/apps/redis_node_${port}/conf/redis_${port}.conf << EOF
  # 加载原有配置文件
  include /etc/redis/redis.conf
  # 端口
  port ${port} 
  bind 0.0.0.0
  protected-mode no
  # 启用集群模式
  cluster-enabled yes
  cluster-config-file nodes_${port}.conf
  # 超时时间
  cluster-node-timeout 5000
  # 集群连接地址及端口
  cluster-announce-ip 192.168.11.101
  cluster-announce-port ${port}
  cluster-announce-bus-port 1${port}
  appendonly yes
  # 集群加密
  masterauth 123456
  requirepass 123456
  EOF
  done
  ```

  > `cluster-announce-ip`：集群节点 IP，填写宿主机的 IP
  >
  > `cluster-announce-port`：集群节点映射端口
  >
  > `cluster-announce-bus-port`：集群节点总线端口
  >
  > 这几个参数在创建 Redis Cluster 集群时，容器内记录集群信息的 nodes.conf 文件中会识别到这几个参数代表的该 redis 结点的 ip，port 和集群通信 port。
  >
  > 如果不加这几个参数，集群就是默认的 redis 容器的 ip 和容器内的 6379 和 16379 端口，这样就会造成 pod 重启导致的 ip 变更集群失效和无法在 k8s 集群外访问 redis 容器的问题。

- 创建容器并启动

  ```bash
  # 循环创建节点
  $ for port in `seq 6381 6383`; do
    docker run -d --name redis_node_${port} --restart=always --privileged=true \
    -v /tmp/redis.conf:/etc/redis/redis.conf \
    -v /tmp/apps/redis_node_${port}/conf/redis_${port}.conf:/etc/redis/redis_${port}.conf \
    -v /tmp/apps/redis_node_${port}/data:/data \
    --net=host\
    redis redis-server /etc/redis/redis_${port}.conf
  done
  ```

**集群配置**

- 进入容器中构建主从关系

  ```bash
  $ docker exec -it redis_node_6381 /bin/bash
  $ redis-cli -p 6391 -a 123456 --cluster create 192.168.11.101:6381 192.168.11.101:6382 192.168.11.101:6383 192.168.11.102:6381 192.168.11.102:6382 192.168.11.102:6383 --cluster-replicas 1
  ```

- 查看集群信息（ip 可以是任何一个节点）

  ```bash
  $ redis-cli --cluster check 192.168.11.101:6381
  ```
  
- 查看集群状态

  ```bash
  $ redis-cli -p 6381 -a 123456
  $ cluster info
  $ cluster nodes
  ```

**测试集群**

使用`redis-cli -c`连接到集群上，`set`一个值，然后从其他节点再获取值查看是否成功

```bash
# 客户端连接，-c 代表连接集群
$ redis-cli -c -p 6381 -a 123456
```

**spring boot 配置**

```yaml
spring:
  redis:
    password: 123456
    cluster:
      nodes: 192.168.3.13:6381,192.168.3.13:6382,192.168.3.13:6383,192.168.3.14:6381,192.168.3.14:6382,192.168.3.14:6383
```

### 3.4 案例

**主从容错切换**

主：6381，从：6384

- 先停止主机 6381
- 再次查看集群信息，发现 6384 上位成为了新的 master
- 启动主机 6381 后 6381 成为了 6384 的从机
- 还原之前的 3 主 3 从
  - 停止 6384，使 6381 成为主机
  - 再启动 6384，6384 成为 6381 的从机

**主从扩容**

- 新建 6387、6388 两个节点+新建后启动+查看是否 8 节点

  ```bash
  $ for port in `seq 6387 6388`; do
    docker run -d --name redis_node_${port} --restart=always --privileged=true\
    -v /tmp/redis.conf:/etc/redis/redis.conf\
    -v /tmp/apps/redis_node_${port}/conf/redis_${port}.conf:/etc/redis/redis_${port}.conf\
    -v /tmp/apps/redis_node_${port}/data:/data\
    --net=host \
    redis redis-server /etc/redis/redis_${port}.conf
  done
  ```

- 进入 6387 容器实例内部，将新增的 6387 节点作为 master 节点加入原集群

  ```bash
  $ docker exec -it redis_node_6387 /bin/bash
  $ redis-cli --cluster add-node 自己实际IP地址:6387 自己实际IP地址:6381
  ```

  6387 就是将要作为master新增节点，6381 就是原来集群节点里面的任意节点

- 检查集群情况

  ```bash
  redis-cli --cluster check 真实ip地址:6381
  ```

- 重新分派槽号

  ```bash
  $ redis-cli --cluster reshard IP地址:端口号
  $ redis-cli --cluster reshard 192.168.111.147:6381
  ```

  选择想要移动多少个槽位 - 选择谁来接受（新节点） - 选择从哪里移出（全部 - all）

- 检查集群情况，发现从 6381/6382/6383 三个旧节点分别匀出 1364 个坑位给新节点 6387

- 为主节点 6387 分配从节点 6388

  ```bash
  $ redis-cli --cluster add-node ip:slave端口 ip:master端口 --cluster-slave --cluster-master-id 新主机节点ID
  $ redis-cli --cluster add-node 192.168.111.147:6388 192.168.111.147:6387 --cluster-slave --cluster-master-id e4781f644d4a4e4d4b4d107157b9ba8144631451
  ```

- 检查集群情况，发现添加从节点成功

**主从缩容**

- 先从集群中将 6387 从节点 6388 删除

  ```bash
  $ redis-cli --cluster del-node ip:从机端口 从机6388节点ID
  $ redis-cli --cluster del-node 192.168.11.101:6388 5d149074b7e57b802287d1797a874ed7a1a284a8
  ```

- 检查一下发现，6388 被删除了，只剩下 7 台机器了

  ```bash
  redis-cli --cluster check 192.168.11.101:6382
  ```

- 将 6387 的槽号清空，重新分配，本例将清出来的槽号都给 6381

  ```bash
  # 这里的 ip 可以是集群中的任意一个节点
  redis-cli --cluster reshard 192.168.11.101:6381
  ```

  选择想要移动多少个槽位 - 选择谁来接受（新节点） - 选择从哪里移出（全部 - all）

- 检查集群情况发现，4096 个槽位都指给 6381，它变成了 8192 个槽位

- 将 6387 删除

  ```bash
  $ redis-cli --cluster del-node ip:端口 6387节点ID
  $ redis-cli --cluster del-node 192.168.11.101:6387 e4781f644d4a4e4d4b4d107157b9ba8144631451
  ```



