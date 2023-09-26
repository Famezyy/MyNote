**简介**

> 直接通过内存读取，减少 IO 操作，降低 CPU 及内存压力
>
> NoSql：not only SQL，泛指非关系型数据，以简单的 key-value 形式存储
>
> - 不遵循 SQL 标准
> - 不支持 ACID
> - 远超于 SQL 性能
>
> 使用场景：
>
> - 高并发
> - 海量数据读写
> - 对数据高扩展性
>
> 不适用于：
>
> - 需要事务支持
> - 基于 SQL 的结构化查询

**常见的 NoSql：**

1. Memcache

- 数据都在内存中，一般不持久化
- 支持类型单一，简单的 key-value 模式
- 一般作为缓存数据库辅助持久化的数据库

2. Redis

- 几乎覆盖了 Memcache 的绝大部分功能
- 数据都在内存中，支持持久化，主要用作数据备份
- 支持多种数据结构存储，如 list、set、hash、zset 等
- 一般作为缓存数据库辅助持久化的数据库

3. MongoDB

- 高性能，开源，模式自由的文档型数据库
- 数据都在内存中，可以把不常用的数据保存到硬盘
- 对 value 提供了丰富的查询功能
- 支持二进制数据及大型对象
- 可以根据数据的特点替代 RDBMS，成为独立的数据库

---

## 一、概述、安装与启动

- Redis 是一个开源的 key-value 存储系统
- 和 Memcached 类似，它支持存储的 value 类型相对更多，包括 sting、list、set、zset 和 hash
- 这些数据类型都支持 push/pop，add/remove 及取交集并集和差集及其他操作，而且这些操作都是原子性的
- 在此基础上，Redis 支持各种不同方式的排序
- 与 memcached 一样，数据都是缓存在内存中
- 区别的是 Redis 会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件

**安装：**

- [Redis下载](https://download.redis.io/releases/redis-6.2.3.tar.gz?_ga=2.17544240.809792253.1621322316-754193776.1621322316)

- 将压缩包上传到Linux服务器上 /opt 目录下

- 解压压缩文件：`tar -zxvf redis-6.2.3.tar.gz`

- 安装 GCC 编译器（需要 C 语言环境）：`yum install -y gcc`

- 查看 GCC 版本：`gcc --version`

- 进入解压后的 redis 文件夹：`cd /opt/redis-6.2.3`

- 在redis-6.2.3目录下执行 `make` 命令
- 在redis-6.2.3目录下执行 `make install` 命令

- 若指定安装目录，可将上两步替换为以下命令，将安装目录修改为自己的：`make PREFIX=/usr/local/redis install`

- 默认安装目录：`/usr/local/bin`

安装出错时：

- 重新安装 GCC 编译器：

  ```
  sudo apt-get update
  sudo apt-get remove gcc
  sudo apt-get install gcc
  ```

- 执行命令 `make distclean` 后重新 `make`

查看默认安装目录：

- redis-benchmark：性能测试工具

- redis-check-aof：修复有问题的 APF 文件

- redis-check-dump：修复有问题的 dump.rdb 文件

- redis-sentinel：Redis集群使用

- **redis-sercer**：Redis 服务器启动命令

- **redis-cli**：客户端，操作入口

**启动：**

1. 前台启动(不推荐使用)

```
cd /usr/local/bin
redis-server
```

2. 后台启动

- 拷贝一份 redis.conf 到其他目录

  ```
  cp /opt/redis-6.2.6/redis.conf /etc/redis.conf
  ```

- **设置 redis.conf 的 [daemonize](#daemonize) 值为 yes**

  ```
  vi /etc/redis.conf
  ```

- 若用阿里云安装的 redis，必须设置密码，不然会被挖矿

  - **修改配置文件，设置 requirepass 值，即密码值**

    ```
    requirepass xxx
    ```

- redis启动

  ```
  cd /usr/local/bin
  redis-server /etc/redis.conf
  ```

- 客户端访问

  - 无密码

    ```
    redis-cli
    ```

  - 有密码

    ```
    redis-cli -a 密码
    ```

    或者使用 redis-cli 进入 redis 后，使用 auth "密码" 认证

- <p name="shutdown">redis关闭</p>

  ```
  redis-cli shutdown
  ```

---

## 二、相关知识介绍

**端口 6379 从何而来** —— Alessia **Merz**

> - 默认 16 个数据库，类似数组下标从 0 开始，初始默认使用 0 号库
> - 使用 `select <dbid>` 来切换数据库，如：select 1
> - 统一密码管理，所有库密码相同
> - `dbsize` 查看当前数据库的 key 数量
> - `flushdb` 清空当前库
> - `flushall` 清空全部库

> - Redis 是单线程 + 多路 IO 复用技术
>
> - 多路复用是指使用一个线程来检查多个文件描述符（Socket）的就绪状态，比如调用 select 和 poll 函数，传入多个文件描述符，如果有一个文件描述符就绪则返回，否则阻塞直到超时。得到就绪状态后进行真正的操作可以在同一个线程里执行，也可以启动线程执行（比如使用线程池）
>
> - 串行 VS 多线程 + 锁（Memcached） VS 单线程 + 多路 IO 复用（Redis）
>
> - 与 Memcache 三点不同：
>
>   - 支持多数据类型
>
>   - 支持持久化
>
>   - **单线程 + 多路 IO 复用**
>
>     <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220114005411353-34b19f6c85dd65cd4a7c75272fd3ca53-1d5764.png" alt="image-20220114005411353" style="zoom:67%;" />

---

## 三、常用的Key操作

- `set k1 a`：key:k1；value:a

- `keys *`：查看当前库所有key
- `exists key`：判断某个 key 是否存在，例：exists k1
- `type key`：查看你的 key 是什么类型，例：type k1 
- `del key`：删除指定的 key 数据，例：del k1 
- `unlink key`：根据 value 选择非阻塞删除仅将 keys 从 keyspace 元数据中删除，真正的删除会在后续异步操作
- `expire key <seconds>`：为给定的key设置过期时间：expire k2 10 
- `ttl key`：查看还有多少秒过期，-1 表示永不过期，-2 表示已过期或不存在的 key，例：ttl k2 	
- `flushdb`：清空当前库

---

## 四、常用数据类型

### 1.字符串——String

#### 1.1 简介

- String 是 Redis 最基本的类型，一个 key 对应一个 value
- String 类型是二进制安全的，以为着 String 可以包含任何数据，比如 jpg 图片 或序列化的对象
- 一个 Redis 中字符串 value 最多可以是 512M

#### 1.2 常用命令

<style>
    td{
        font-weight:normal;
    }
</style>
<table style="text-align:center;">
    <tr>
    	<th>命令</th>
        <th>描述</th>
    </tr>
    <tr>
    	<td>set &ltkey>&ltvalue></td>
        <td>添加键值对，若 key 值存在，则会覆盖之前的值</td>
    </tr>
    <tr>
    	<td>get &ltkey></td>
        <td>查询对应键值</td>
    </tr>
    <tr>
    	<td>append &ltkey>&ltvalue></td>
        <td>将给定的 value 追加到原值的末尾</td>
    </tr>
    <tr>
    	<td>strlen &ltkey></td>
        <td>获得值的长度</td>
    </tr>
    <tr>
    	<td>setnx &ltkey>&ltvalue></td>
        <td>只有当 key 不存在时，设置 key 的值</td>
    </tr>
    <tr>
    	<td>incr &ltkey></td>
        <td>将 key 中存储的数字值增加 1，只能对数字操作，如果为空，新增值为 1</td>
    </tr>
    <tr>
    	<td>decr &ltkey></td>
        <td>将 key 中存储的数字值减 1，只能对数字操作，如果为空，新增值为 -1</td>
    </tr>
    <tr>
    	<td>incrby / decryby &ltkey>&lt步长></td>
        <td>将 key 中储存的数字值增减，自定义步长</td>
    </tr>
    <tr>
        <td>mset &ltkey1>&ltvalue1>&ltkey2>&ltvalue2>...</td>
    	<td>同时设置一个或多个键值对</td>
    </tr>
    <tr>
    	<td>mget &ltkey1>&ltvalue1>&ltkey2>&ltvalue2>...</td>
        <td>同时获取一个或多个键值对</td>
    </tr>
	<tr>
    	<td>msetnx &ltkey1>&ltvalue1>&ltkey2>&ltvalue2>...</td>
        <td>同时设置一个或多个键值对，当且仅当所有给定 key 都不存在</br><span style="color:#FF0000;">注意：</span>具有原子性，有一个失败则都失败</td>
	</tr>
	<tr>
		<td>getrange &ltkey>&lt起始位置>&lt结束位置></td>
        <td>获得值的范围（包含起始终止位置）</td>
	</tr>
	<tr>
		<td>setrange &ltkey>&lt起始位置>&ltvalue></td>
        <td>用 &ltvalue> 复写 &ltkey> 所存储的字符串值，从 &lt起始位置> 开始（索引从 0 开始）</td>
	</tr>
    <tr>
        <td>setex &ltkey>&lt过期时间>&ltvalue></td>
        <td>设置键值的同时，设置过期时间，单位秒</td>
    </tr>
	<tr>
		<td>getset &ltkey>&ltvalue></td>
        <td>设置新值同时获得旧值</td>
	</tr>
</table>

> **原子性**
>
> - INCR Key：对存储在指定 key 的数值进行原子的加 1 操作
>
> - Redis 单命令的原子性得益于 Redis 的单线程
>
> **扩展**
>
> JAVA 中的 i++ 不是原子操作
>
> - 取值
> - 运算
> - 写入
>
> 两个线程分别对 i 进行 ++100 次，值可能为 2~200

#### 1.3 数据结构

> ​    String 的数据结构为简单动态字符串（Simple Dynamic String，缩写 SDS），是可以修改的字符串，内部结构实现上类似于 JAVA 的 ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配。
>
> <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220114015557740-5bafa158532915b834939cf25093b9a5-de5e51.png" alt="image-20220114015557740" style="zoom:80%;" />
>
> ​    如图所示，内部为当前字符串实际分配的空间 capacity 一般要高于实际字符串长度，当字符串长度小于 1M 时，扩容都是加倍现有的空间，如果超过 1M，扩容是一次只会多扩容 1M 的空间，需要注意的是字符串最大长度为 512M。

### 2.列表——List

#### 2.1 简介

- 单键多值
- Redis 列表是简单的字符串列表，按照插入顺序排序，可以添加一个元素到列表的头部（左边）或尾部（右边）
- 它的底层实现是双向链表，对两端的操作性能很高，通过索引下标的操作中间的节点性能较差

#### 2.2 常用命令

|                        命令                        |                             描述                             |
| :------------------------------------------------: | :----------------------------------------------------------: |
|  lpush / rpush `<key><value1><value2><value3>` …   |      从左或者右 **依次** 插入一个或者多个值(头插与尾插)      |
|               lpop / rpop `<key><n>`               | 从左或者右吐出一个或者多个值(值在键在,值都没,键都没)<br />没有值时返回 (nil) |
|              rpoplpush `<key1><key2>`              |      从`<key1>`列表右边吐出一个值，插到`<key2>`列表左边      |
|            lrange `<key><start><stop>`             | 按照索引下标获取元素(从左到右)<br />0左边第一个，-1右边第一个，（0 -1表示获取所有） |
|               lindex `<key><index>`                |     按照索引下标获得元素(从左到右)，获取不到时返回 (nil)     |
|                    llen `<key>`                    |                         获取列表长度                         |
| linsert `<key>` before / after `<value><newvalue>` |        在`<value>`的 **前面 / 后面** 插入`<newvalue>`        |
|               lrem`<key><n><value>`                | 从左边删除 n 个与 value 同样的值 (从左到右)，返回删除成功的个数 |
|             lset`<key><index><value>`              |           在列表 key 中的下标 index 中修改值 value           |

#### 2.3 数据结构

> - List 的数据结构为快速链表 quickList
>
> - 首先在列表元素较少的情况下会使用一块连续的内存存储，这个结构是 ziplist，也即是压缩列表
>
> - 它将所有的元素紧挨着一起存储，分配的是一块连续的内存
>
> - 当数据量比较多的时候才会改成 quicklist
>
> - 因为普通的链表需要的附加指针空间太大，会比较浪费空间。比如这个列表里存的只是 int 类型的数据，结构上还需要两个额外的指针 prev 和 next
>
>   <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/88b974e67c474683a08e4a19fa28bd8b-ff58ddb40c2a34748f29ddbafb26631d-84c6c2.png" alt="img" style="zoom:80%;" />
>
> - Redis 将链表和 ziplist 结合起来组成了 quicklist。也就是将多个 ziplist 使用双向指针串起来使用。这样既满足了快速的插入删除性能，又不会出现太大的空间冗余

### 3.集合——Set

#### 3.1 简介

- Redis set 对外提供的功能与list类似是一个列表的功能，特殊之处在于 set 是可以自动排重的，当你需要存储一个列表数据，又不希望出现重复数据时，set 是一个很好的选择，并且 set 提供了判断某个成员是否在一个 set 集合内的重要接口，这个也是 list 所不能提供的

- Redis 的 Set 是 string 类型的无序集合。它底层其实是一个 value 为 null 的 hash 表，所以添加，删除，查找的复杂度都是 O(1)

- 一个算法，随着数据的增加，执行时间的长短，如果是 O(1)，数据增加，查找数据的时间不变

#### 3.2 常用命令

|                命令                |                             描述                             |
| :--------------------------------: | :----------------------------------------------------------: |
|   sadd `<key><value1><value2>` …   | 将一个或多个 member 元素加入到集合 key 中，已经存在的 member 元素将被忽略 |
|          smembers `<key>`          |                      取出该集合的所有值                      |
|      sismember `<key><value>`      |     判断集合`<key>`是否为含有该`<value>`值，有 1，没有 0     |
|            scard`<key>`            |                     返回该集合的元素个数                     |
|   srem `<key><value1><value2>` …   | 删除集合中的某些元素，返回删除成功的个数，自动忽略不存在的元素 |
|            spop `<key>`            |                 **随机从该集合中吐出一个值**                 |
|       srandmember `<key><n>`       |          随机从该集合中取出n个值。不会从集合中删除           |
| smove `<source><destination>`value |           把集合中一个值从一个集合移动到另一个集合           |
|       sinter `<key1><key2>`        |                    返回两个集合的交集元素                    |
|       sunion `<key1><key2>`        |                    返回两个集合的并集元素                    |
|        sdiff `<key1><key2>`        |    返回两个集合的 **差集** 元素(key1中的，不包含key2中的)    |

#### 3.2 数据结构

> - Set 数据结构是 dict 字典，字典是用哈希表实现的。
>
> - Java中 HashSet 的内部实现使用的是 HashMap，只不过所有的 value 都指向同一个对象。Redis 的 set 结构也是一样，它的内部也使用 hash 结构，所有的 value 都指向同一个内部值。

### 4.哈希——Hash

#### 4.1 简介

- Redis hash 是一个键值对集合

- Redis hash 是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象

- 类似 Java 里面的 Map<String,Object>

- 用户 ID 为查找的 key，存储的 value 用户对象包含姓名，年龄，生日等信息，如果用普通的 key/value 结构来存储主要有以下 2 种存储方式：

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,size_16,color_FFFFFF,t_70-413c16bc5fbb4bd7c8a7eac17cc6defe-d93d2a" alt="img" style="zoom: 80%;" />

#### 4.2 常用命令

|                      命令                       |                             描述                             |
| :---------------------------------------------: | :----------------------------------------------------------: |
|           hset `<key><field><value>`            |        给 key 集合中的 field 键赋值 value，可重复设置        |
|               hget `<key><field>`               |                 从 key 集合 field 取出 value                 |
| hmset `<key1><field1><value1><field2><value2>`… |                      批量设置 hash 的值                      |
|             hexists`<key1><field>`              |    查看哈希表中，给定域 field 是否存在，存在 1，不存在 0     |
|                  hkeys `<key>`                  |                 列出该 hash 集合的所有 field                 |
|                  hvals `<key>`                  |                 列出该 hash 集合的所有 value                 |
|        hincrby `<key><field><increment>`        | 为哈希表 key 中的域 field 的值加上增量 increment，可以为负数 |
|          hsetnx `<key><field><value>`           | 将哈希表 key 中的 field 的值设置为 value，当且仅当域不存在，若存在则不会添加成功 |

#### 4.3 数据结构

> Hash 类型对应的数据结构是两种：ziplist（压缩列表），hashtable（哈希表）。当 field-value 长度较短且个数较少时，使用 ziplist，否则使用 hashtable

### 5.有序集合——Zset

#### 5.1 简介

- Redis有序集合 zset 与普通集合 set 非常相似，是一个没有重复元素的字符串集合。

- 不同之处是有序集合的每个成员都关联了一个评分（score）,这个评分（score）被用来按照从最低分到最高分的方式排序集合中的成员。集合的成员是唯一的，但是评分可以是重复了 。

- 因为元素是有序的, 所以你也可以很快的根据评分（score）或者次序（position）来获取一个范围的元素。

- 访问有序集合的中间元素也是非常快的,因此你能够使用有序集合作为一个没有重复成员的智能列表。

#### 5.2 常用命令

|                             命令                             |                             描述                             |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
|        zadd `<key><score1><value1><score2><value2>`…         |  将一个或多个 member 元素及其 score 值加入到有序集 key 当中  |
|            zrange`<key><start><stop>`[withscores]            | 返回有序集 key 中，下标在`<start><stop>`之间的元素<br/>加上 withscores 可以让分数一起和值返回到结果集 |
| zrangebyscore`<key><min><max>`[withscores] [limit offset count] | 返回有序集 key 中，所有 score 值介于 min 和 max 之间（包括等于 min 或 max）的成员<br />有序集成员按 score 值递增（从小到大）次序排列，可以使用 limit 进行分页 |
| zrevrangebyscore`<key><max><min>`[withscores] [limit offset count] |                    同上，改为从大到小排列                    |
|              zincrby`<key><increment><member>`               |            为`member`的 score 加上增量 increment             |
|                     zrem`<key><member>`                      |                   删除该集合下指定值的元素                   |
|                   zcount`<key><min><max>`                    |              统计该集合指定分数区间内的元素个数              |
|                     zrank`<key><member>`                     |           返回该`member`在集合中的排名，从 0 开始            |

#### 5.3 数据结构

> - SortedSet(zset)是Redis提供的一个非常特别的数据结构，一方面它等价于Java的数据结构Map<String,Double>，可以给每一个元素value赋予一个权重score，另一方面它又类似于TreeSet，内部的元素会按照权重score进行排序，可以得到每个元素的名次，还可以通过score的范围来获取元素的列表
>
> - zset底层使用了两个数据结构
>
>   - hash，hash 的作用就是关联元素 value 和权重 score，保障元素 value 的唯一性，可以通过元素 value 找到相应的 score 值
>
>   - 跳跃表，跳跃表的目的在于给元素 value 排序，根据 score 的范围获取元素列表

#### 5.4 跳跃表

>**简介**
>
>  有序集合在生活中比较常见，例如根据成绩对学生排名，根据得分对玩家排名等。对于有序集合的底层实现，可以用数组、平衡树、链表等。数组不便元素的插入、删除；平衡树或红黑树虽然效率高但结构复杂；链表查询需要遍历所有效率低。Redis 采用的是跳跃表。跳跃表效率堪比红黑树，实现远比红黑树简单。
>
>**实例**
>
>对比有序链表和跳跃表，从链表中查询出51
>
>- 有序链表
>
>  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/27a06fae49d54080a4808cf5d6a11c06-a765c3f9233f7fcac80f56a734e1544a-ace44a.png" alt="img" style="zoom:80%;" />
>
>  要查找值为51的元素，需要从第一个元素开始依次查找、比较才能找到。共需要6次比较
>
>- 跳跃表
>
>  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/ad5021fa573549748ed6809b4b83f316-7e2df50256227fc6307642ff155b2a13-4242ac.png" alt="img" style="zoom:80%;" />
>
>  - 从第2层开始，1节点比51节点小，向后比较
>
>  - 21节点比51节点小，继续向后比较，后面就是NULL了，所以从21节点向下到第1层
>
>  - 在第1层，41节点比51节点小，继续向后，61节点比51节点大，所以从41向下
>
>  - 在第0层，51节点为要查找的节点，节点被找到，共查找4次。
>
>**从此可以看出跳跃表比有序链表效率要高**

---

## 五、配置文件

- **Units单位**

  > - 配置大小单位,开头定义了一些基本的度量单位，只支持 bytes 字节类型，不支持 bit
  >
  > - 大小写不敏感
  >
  > ```text
  > # 1k => 1000 bytes
  > # 1kb => 1024 bytes
  > # 1m => 1000000 bytes
  > # 1mb => 1024*1024 bytes
  > # 1g => 1000000000 bytes
  > # 1gb => 1024*1024*1024 bytes
  > # units are case insensitive so 1GB 1Gb 1gB are all the same.
  > ```
  >
  
- **INCLUDES包含**

  > 类似 jsp 中的 include，多实例的情况可以把公用的配置文件提取出来
  >
  > ```text
  > # If instead you are interested in using includes to override configuration
  > # options, it is better to use include as the last line.
  > #
  > # include /path/to/local.conf
  > # include /path/to/other.conf
  > ```

- **网络相关配置**

  > 1. bind
  >
  >    - 默认情况 bind=127.0.0.1 只能接受本机的访问请求
  >
  >    - 不写的情况下，无限制接受任何 ip
  >
  >    - 生产环境肯定要写你应用服务器的地址；服务器是需要远程访问的，所以需要将其注释掉
  >
  >    - 如果开启了 protected-mode，那么在没有设定 bind ip 且没有设密码的情况下，Redis 只允许接受本机的响应
  >
  > ```text
  > # Examples:
  > #
  > # bind 192.168.1.100 10.0.0.1     # listens on two specific IPv4 addresses
  > # bind 127.0.0.1 ::1              # listens on loopback IPv4 and IPv6
  > # bind * -::*                     # like the default, all available interfaces
  > bind 127.0.0.1 -::1
  > ```
  >
  > 2. protected-mode
  >
  >    将本机访问保护模式设置 no，即允许远程访问
  >
  > ```text
  > # The server only accepts connections from clients connecting from the
  > # IPv4 and IPv6 loopback addresses 127.0.0.1 and ::1, and from Unix domain
  > # sockets.
  > #
  > # By default protected mode is enabled. You should disable it only if
  > # you are sure you want clients from other hosts to connect to Redis
  > # even if no authentication is configured, nor a specific set of interfaces
  > # are explicitly listed using the "bind" directive.
  > protected-mode yes
  > ```
  >
  > 3. PORT
  >
  >    端口号
  >
  >
  > ```text
  > # Accept connections on the specified port, default is 6379 (IANA #815344).
  > # If port 0 is specified Redis will not listen on a TCP socket.
  > port 6379
  > ```
  >
  > 4. tcp-backlog
  >    - 设置 tcp 的 backlog，backlog 其实是一个连接队列，backlog 队列总和=未完成三次握手队列 + 已经完成三次握手队列。
  >
  >    - 在高并发环境下你需要一个高backlog值来避免慢客户端连接问题。
  >
  >    - 注意 Linux 内核会将这个值减小到 /proc/sys/net/core/somaxconn 的值（128），所以需要确认增大 /proc/sys/net/core/somaxconn 和 /proc/sys/net/ipv4/tcp_max_syn_backlog（128）两个值来达到想要的效果
  >
  >
  > ```text
  > # In high requests-per-second environments you need a high backlog in order
  > # to avoid slow clients connection issues. Note that the Linux kernel
  > # will silently truncate it to the value of /proc/sys/net/core/somaxconn so
  > # make sure to raise both the value of somaxconn and tcp_max_syn_backlog
  > # in order to get the desired effect.
  > tcp-backlog 511
  > ```
  >
  > 5. timeout
  >
  >    一个空闲的客户端维持多少秒会关闭，0 表示关闭超时功能，即永不关闭
  >
  >
  > ```text
  > # Close the connection after a client is idle for N seconds (0 to disable)
  > timeout 0
  > ```
  >
  > 6. tcp-keepalive
  >
  >    - 对访问客户端的一种心跳检测，每隔 n 秒检测一次
  >
  >
  >    - 单位为秒，如果设置为 0，则不会进行 Keepalive 检测，建议设置成 60
  >
  >
  > ```text
  > # If non-zero, use SO_KEEPALIVE to send TCP ACKs to clients in absence
  > # of communication. This is useful for two reasons:
  > #
  > # 1) Detect dead peers.
  > # 2) Force network equipment in the middle to consider the connection to be
  > #    alive.
  > #
  > # On Linux, the specified value (in seconds) is the period used to send ACKs.
  > # Note that to close the connection the double of the time is needed.
  > # On other kernels the period depends on the kernel configuration.
  > tcp-keepalive 300
  > ```

- **GENERAL通用**

  > 1. <p name="daemonize">daemonize</p>
  >
  >    - 是否为后台进程，设置为 yes
  >    - 守护进程，后台启动
  >    
  >
  > ```text
  > # By default Redis does not run as a daemon. Use 'yes' if you need it.
  > # Note that Redis will write a pid file in /var/run/redis.pid when daemonized.
  > # When Redis is supervised by upstart or systemd, this parameter has no impact.
  > daemonize yes
  > ```
  >
  > 2. pidfile
  >    - 存放 pid 文件的位置，每个实例会产生一个不同的 pid 文件
  >
  > ```text
  > # If a pid file is specified, Redis writes it where specified at startup
  > # and removes it at exit.
  > #
  > # When the server runs non daemonized, no pid file is created if none is
  > # specified in the configuration. When the server is daemonized, the pid file
  > # is used even if not specified, defaulting to "/var/run/redis.pid".
  > #
  > # Creating a pid file is best effort: if Redis is not able to create it
  > # nothing bad happens, the server will start and run normally.
  > #
  > # Note that on modern Linux systems "/run/redis.pid" is more conforming
  > # and should be used instead.
  > pidfile /var/run/redis_6379.pid
  > ```
  >
  > 3. loglevel
  >
  >    - 指定日志记录级别，Redis总共支持四个级别：debug、verbose、notice、warning，默认为 notice
  >
  >    - 四个级别根据使用阶段来选择，生产环境选择 notice 或者 warning
  >
  > ```text
  > # Specify the server verbosity level.
  > # This can be one of:
  > # debug (a lot of information, useful for development/testing)
  > # verbose (many rarely useful info, but not a mess like the debug level)
  > # notice (moderately verbose, what you want in production probably)
  > # warning (only very important / critical messages are logged)
  > loglevel notice
  > ```
  >
  > 4. logfile
  >
  >    日志文件名称
  >
  > ```text
  > # Specify the log file name. Also the empty string can be used to force
  > # Redis to log on the standard output. Note that if you use standard
  > # output for logging but daemonize, logs will be sent to /dev/null
  > logfile ""
  > ```
  >
  > 5. databases
  >
  >    设定库的数量 默认 16，默认数据库为 0，可以使用SELECT <dbid> 命令在连接上指定数据库 id
  >
  > ```text
  > # Set the number of databases. The default database is DB 0, you can select
  > # a different one on a per-connection basis using SELECT <dbid> where
  > # dbid is a number between 0 and 'databases'-1
  > databases 16
  > ```

- **SECURITY安全**

  > **requirepass**
  >
  > - 访问密码的查看、设置和取消
  >
  > - 在命令中设置密码，只是临时的。重启 redis 服务器，密码就还原了。
  >
  > - 永久设置，需要再配置文件中进行设置
  >
  > ````text
  > requirepass zhaoyouyi1993919
  > ````
  > 
  > <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,size_16,color_FFFFFF,t_702-4c0720650e2c246c146215bf97691293-3cf900" alt="img" style="zoom:80%;" />
  
- **LIMITS限制**

  > 1. maxclients
  >
  >    - 设置 redis 同时可以与多少个客户端进行连接
  >
  >    - 默认情况下为 10000 个客户端。
  >
  >    - 如果达到了此限制，redismax number of clients reached
  >
  > ```text
  > maxclients 10000
  > ```
  > 
  > 2. maxmemory
  > 
  >    - 建议必须设置，否则，将内存占满，造成服务器宕机
  > 
  >    - 设置 redis 可以使用的内存量。一旦到达内存使用上限，redis 将会试图移除内部数据，移除规则可以通过 maxmemory-policy 来指定
  > 
  >    - 如果 redis 无法根据移除规则来移除内存中的数据，或者设置了“不允许移除”，那么 redis 则会针对那些需要申请内存的指令返回错误信息，比如 SET、LPUSH 等
  > 
  >    - 但是对于无内存申请的指令，仍然会正常响应，比如 GET 等。如果你的 redis 是主 reds（说明你的redis有从redis），那么在设置内存使用上限时，需要在系统中留出一些内存空间给同步队列缓存，只有在你设置的是“不移除”的情况下，才不用考虑这个因素
  > 
  > ```text
  > # maxmemory <bytes>
  > ```
  >
  > 3. maxmemory-policy
  >
  >    - noeviction：默认策略，不淘汰数据；大部分写命令都将返回错误（DEL 等少数除外）
  >
  >    - volatile-lru：使用 LRU 算法移除 key，只对设置了过期时间的键（最近最少使用）
  >
  >    - allkeys-lru：在所有集合 key 中，使用 LRU 算法移除 key
  >
  >    - volatile-random：在过期集合中移除随机的 key，只对设置了过期时间的键
  >
  >    - allkeys-random：在所有集合 key 中，移除随机的 key
  > 
  >    - volatile-ttl：移除那些 TTL 值最小的 key，即那些最近要过期的 key
  > 
  >    - noeviction：不进行移除，针对写操作，只是返回错误信息
  > 
  >    - allkeys-lfu：从所有数据中根据 LFU 算法挑选数据淘汰（4.0 及以上版本可用）
  > 
  >    - volatile-lfu：从设置了过期时间的数据中根据 LFU 算法挑选数据淘汰（4.0 及以上版本可用）
  > 
  > ```text
  > # maxmemory-policy noeviction
  > ```
  > 
  > 4. maxmemory-samples
  > 
  >    - 设置样本数量，LRU 算法和最小 TTL 算法都并非是精确的算法，而是估算值，所以你可以设置样本的大小，redis 默认会检查这么多个 key 并选择其中 LRU 的那个
  > 
  >    - 一般设置 3 到 7 的数字，数值越小样本越不准确，但性能消耗越小
  > 
  > ```text
  > # LRU, LFU and minimal TTL algorithms are not precise algorithms but approximated
  > # algorithms (in order to save memory), so you can tune it for speed or
  > # accuracy. By default Redis will check five keys and pick the one that was
  > # used least recently, you can change the sample size using the following
  > # configuration directive.
  >#
  > # The default of 5 produces good enough results. 10 Approximates very closely
  ># true LRU but costs more CPU. 3 is faster but not very accurate.
  > #
  ># maxmemory-samples 5
  > 
  >```

---

## 六、Redis的发布和订阅

> 什么是发布和订阅？
>
> - Redis 发布订阅 (pub/sub) 是一种消息通信模式：发送者 (pub) 发送消息，订阅者 (sub) 接收消息
> - Redis 客户端可以订阅任意数量的频道

1. 客户端可以订阅频道如下图

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/d6d85c97cd934b26829090da30d8a928-dd5e1cf8891ec4e1ed4f6d16b9f550fb-dd53c5.png" alt="在这里插入图片描述"  />

2. 当给这个频道发布消息后，消息就会发送给订阅的客户端

**演示**：

- 打开一个客户端订阅 channel1：`SUBSCRIBE channel1`

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/a5e4d5496e12497d93d4b5a6c30396bb-39bb248035113f0b107fd5bec81153fd-7386ba.png" alt="img" style="zoom:80%;" />

- 打开另一个客户端，给 channel1 发布消息 hello：`publish channel1 hello`

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/20568f1ac96b4d0491f1745db3d4b4b9-8a30831836e1f5a756ba815199233635-ea11b0.png" alt="img" style="zoom:80%;" />

  > 返回的1是订阅者数量

- 打开第一个客户端可以看到发送的消息

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/c5c868e2eec64f7cbbd879b1c60799df-7b9d8acc50482f194b9bd9fc24e72958-ad1628.png" alt="img" style="zoom:80%;" />

  > 注：发布的消息没有持久化，如果在订阅的客户端收不到 hello，只能收到订阅后发布的消息

---

## 七、Redis新数据类型

### 1. Bitmaps

#### 1.1 简介

> ​    现代计算机用二进制（位） 作为信息的基础单位， 1 个字节等于 8 位， 例如 “abc” 字符串是由 3 个字节组成， 但实际在计算机存储时将其用二进制表示， “abc” 分别对应的 ASCII 码分别是97、 98、 99， 对应的二进制分别是 01100001、 01100010和01100011，如下图
>
> <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/60424e743cb64d28ac26156633efc915-2ba140fe52fb09eb913ee7321ddc9d8b-57b634.png" alt="img" style="zoom:80%;" />
>
> ​    合理地使用操作位能够有效地提高内存使用率和开发效率

Redis提供了 Bitmaps 这个“数据类型”可以实现对位的操作

- Bitmaps本身不是一种数据类型， 实际上它就是 **字符串**（key-value） ， 但是它可以对字符串的位进行操作
- Bitmaps单独提供了一套命令，所以在中使用和使用字符串的方法不太相同，可以把想象成一个以位为单位的数组，数组的每个单元只能存储 0 和 1，数组的下标在中叫做偏移量

#### 1.2 命令

- **setbit**

  `setbit <key><offset><value>`：设置 Bitmaps 中某个偏移量的值（0或1），不是 0 或 1 则报错超过范围

  - offset：偏移量从 0 开始

  - 实例

    - 每个独立用户是否访问过网站存放在 Bitmaps 中， 将访问的用户记做 1， 没有访问的用户记做 0， 用偏移量作为用户的 id

    - 设置键的第 offset 个位的值（从 0 算起） ， 假设现在有20个用户，userid=1， 6， 11， 15， 19 的用户对网站进行了访问， 那么当前 Bitmaps 初始化结果如图

    <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/9070a0aa234e458a92da950e5b337752-e662c75327e362830153513d821c9a5d-76d9cc.png" alt="img" style="zoom:80%;" />

    - users:20000911 代表 2000-09-11 这天的独立访问用户的 Bitmaps

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220115193427992-1019ba5a733593a85af0353132fcee80-0ba8e9.png" alt="image-20220115193427992" style="zoom:80%;" />

> 注：
>
> - 很多应用的用户id以一个指定数字（例如10000） 开头， 直接将用户 id 和 Bitmaps 的偏移量对应势必会造成一定的浪费， 通常的做法是每次做 setbit 操作时将用户 id 减去这个指定数字
>
> - 在第一次初始化 Bitmaps 时， 假如偏移量非常大， 那么整个初始化过程执行会比较慢， 可能会造成 Redis 的阻塞

- **getbit**

  `getbit <key><offset>`：获取 Bitmaps 中某个偏移量的值

  - 实例

    - 获取 id=8 的用户是否在 2000-09-11 这天访问过， 返回 0 说明没有访问过

      <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220115193507925-ef39cd01924df4807c9fc49611728e6f-c5e730.png" alt="image-20220115193507925"  />

      > 注：因为 8 与 12 根本不存在，所以也是返回 0

- **bitcount**

  `bitcount <key><start><end>`：统计字符串被设置为 1 的 bit 数

  >   一般情况下，给定的整个字符串都会被进行计数，通过指定额外的 start 或 end 参数，可以让计数只在特定的位上进行。start 和 end 参数的设置，都可以使用负数值：比如 -1 表示最后一个位，而 -2 表示倒数第二个位，start、end 是指 bit 组的字节的下标数，二者皆包含。

  - 实例

    计算 2000 年 9 月 11 访问量

    <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220115194535400-834739a841205a2f80e016adc4da2df4-86e1ef.png" alt="image-20220115194535400"  />

    start 和 end 代表起始和结束字节数， 下面操作计算用户 id 在第 1 个字节到第 3 个字节之间的独立访问用户数， 对应的范围是 8 ~ 31

    <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220115194553619-72659993b5050ff043b53bab6f67875e-0f32f1.png" alt="image-20220115194553619"  />

  - 举例： K1 【01000001 01000000 00000000 00100001】，对应【0，1，2，3】

    1、bitcount K1 1 2 ： 统计下标 1、2 字节组中 bit=1 的个数，即01000000 00000000 –》bitcount K1 1 2 -》1

    2、bitcount K1 1 3 ： 统计下标 1、2 字节组中 bit=1 的个数，即01000000 00000000 00100001 –》bitcount K1 1 3 --》3

    3、bitcount K1 0 -2 ： 统计下标 0 到下标倒数第 2，字节组中 bit=1 的个数，即01000001 01000000 00000000 –》bitcount K1 0 -2 --》3

    注意：redis 的 setbit 设置或清除的是 bit 位置，而 bitcount 计算的是 byte 位置

- **bitop**

  `bitop and(or/not/xor) <destkey> [key…]`

  - bitop是一个复合操作， 它可以做多个 `<key>`（Bitmaps） 的 and（交集） 、 or（并集） 、 not（非） 、 xor（异或） 操作并将结果保存在 `<destkey>` 中

  - 实例：

    2020-11-04 日访问网站的 userid=1,2,5,9

    setbit unique:users:20201104 1 1

    setbit unique:users:20201104 2 1

    setbit unique:users:20201104 5 1

    setbit unique:users:20201104 9 1

    2020-11-03 日访问网站的 userid=0,1,4,9

    setbit unique:users:20201103 0 1

    setbit unique:users:20201103 1 1

    setbit unique:users:20201103 4 1

    setbit unique:users:20201103 9 1

    **计算出两天都访问过网站的用户数量**

    ```
    bitop and unique:users:and:20201104_03 unique:users:20201103 unique:users:20201104
    (integer 2) # 与运算
    ```

    <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,size_16,color_FFFFFF,t_703-155d9c595f5fb6eb194b7007918c907c-b4cd67" alt="img"  />

#### 1.3 Bitmaps与set对比

假设网站有 1 亿用户， 每天独立访问的用户有 5 千万， 如果每天用集合类型和 Bitmaps 分别存储活跃用户可以得到表：

| set和Bitmaps存储一天活跃用户对比 |                    |                  |                        |
| :------------------------------: | :----------------: | :--------------: | :--------------------: |
|             数据类型             | 每个用户id占用空间 | 需要存储的用户量 |       全部内存量       |
|             集合类型             |        64位        |     50000000     | 64位*50000000 = 400MB  |
|             Bitmaps              |        1位         |    100000000     | 1位*100000000 = 12.5MB |

很明显，这种情况下使用 Bitmaps 能节省很多的内存空间， 尤其是随着时间推移节省的内存还是非常可观的

| set和Bitmaps存储独立用户空间对比 |        |        |       |
| :------------------------------: | :----: | :----: | :---: |
|             数据类型             |  一天  | 一个月 | 一年  |
|             集合类型             | 400MB  |  12GB  | 144GB |
|             Bitmaps              | 12.5MB | 375MB  | 4.5GB |

但 Bitmaps 并不是万金油， 假如该网站每天的独立访问用户很少， 例如只有 10 万（大量的僵尸用户） ， 那么两者的对比如下表所示， 很显然， 这时候使用 Bitmaps 就不太合适了， 因为基本上大部分位都是 0

| set和Bitmaps存储一天活跃用户对比（独立用户比较少） |                    |                  |                        |
| -------------------------------------------------- | ------------------ | ---------------- | ---------------------- |
| 数据类型                                           | 每个userid占用空间 | 需要存储的用户量 | 全部内存量             |
| 集合类型                                           | 64位               | 100000           | 64位*100000 = 800KB    |
| Bitmaps                                            | 1位                | 100000000        | 1位*100000000 = 12.5MB |

### 2. HyperLogLog

#### 2.1 简介

​    在工作当中，我们经常会遇到与统计相关的功能需求，比如统计网站 PV（PageView 页面访问量）,可以使用 Redis 的 incr、incrby 轻松实现。但像 UV（UniqueVisitor，独立访客）、独立 IP 数、搜索记录数等需要去重和计数的问题如何解决？这种求集合中不重复元素个数的问题称为基数问题

​    解决基数问题有很多种方案：

（1）数据存储在 MySQL 表中，使用 distinct count 计算不重复个数

（2）使用 Redis 提供的 hash、set、bitmaps 等数据结构来处理

- 以上的方案结果精确，但随着数据不断增加，导致占用空间越来越大，对于非常大的数据集是不切实际的，能否能够降低一定的精度来平衡存储空间？Redis 推出了 HyperLogLog

- Redis HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定的、并且是很小的

- 在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比，但是因为 HyperLogLog 只会根据输入元素来计算基数，而 **不会储存输入元素本身**，所以 HyperLogLog 不能像集合那样，返回输入的各个元素

什么是基数?

​    比如数据集 {1, 3, 5, 7, 5, 7, 8}，那么这个数据集的基数集为 {1, 3, 5 ,7, 8}, 基数（不重复元素个数）为 5。 基数估计就是在误差可接受的范围内，快速计算基数

#### 2.2 命令

- **pfadd**

  `pfadd <key><element><element>...`：添加指定元素到 HyperLogLog 中，如果执行命令后估计的基数发生变化，则返回 1，否则 0

- **pfcount**
  `pfcount<key><key>...`：计算 HLL 的近似基数，可以计算多个 HLL 的基数的和，比如用 HLL 存储每天的 UV，计算一周的 UV 可以使用 7 天的 UV 合并计算即可

  - 实例

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/c23f0f23825946829d849acc0b150d1b-5adb21e404fee03615784d71c8926669-138e15.png" alt="img" style="zoom:80%;" />

- **pfmerge**

  `pfmerge<destkey><sourcekey><sourcekey>...`：将一个或多个`<sourcekey>`合并后的结果存储在`<destkey>`中，比如每月活跃用户可以使用每天的活跃用户来合并计算可得

  - 实例

    <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/1b845dfaec8e4303b4d8a36b40325b55-93fc75c6cb0b388b9bfaeedfab156257-3580e0.png" alt="img" style="zoom:80%;" />

### 3. Geospatial

#### 3.1 简介

​    Redis 3.2 中增加了对 GEO 类型的支持。GEO，Geographic，地理信息的缩写。该类型，就是元素的 2 维坐标，在地图上就是经纬度。redis 基于该类型，提供了经纬度设置，查询，范围查询，距离查询，经纬度 Hash 等常见操作

#### 3.2 命令

- **geoadd**

  `geoadd<key><longitude><latitude><member> [<longitude><latitude><member>...]`：加地理位置（经度，纬度，名称）

  - 实例

    ```text
    geoadd china:city 121.47 31.23 shanghai
    (integer) 1
    geoadd china:city 106.50 29.53 chongqing 114.05 22.52 shenzhen 116.38 39.90 beijing
    (integer) 3
    ```

    > - 两极无法直接添加，一般会下载城市数据，直接通过 Java 程序一次性导入
    >
    > - 有效的经度从 -180 度到 180 度。有效的纬度从 -85.05112878 度到 85.05112878 度
    >
    > - 当坐标位置超出指定范围时，该命令将会返回一个错误
    > - 已经添加的数据，是无法再次往里面添加的

- **geopos**

  `geopos <key><member>[member...]`：获得指定地区的坐标值

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/9182ade18ffb44cbab39e98abfae98e9-eebfc2618e70d9d2862ef3c3f2dd0447-8f5281.png" alt="img" style="zoom:80%;" />

- **geodist**

  `geodist<key><member1><member2>[m|km|ft|mi]`：获取两个位置之间的直线距离

  获取两个位置之间的直线距离：

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/e3d1223b4e1245d5b13aef5a0f8d57f5-33633dacd1d9357d0a06a95b41f9fcc0-0c02fc.png" alt="img" style="zoom:80%;" />

  > 单位：
  >
  > - m 表示单位为米[默认值]
  >
  > - km 表示单位为千米
  >
  > - mi 表示单位为英里
  >
  > - ft 表示单位为英尺
  >
  > - 如果用户没有显式地指定单位参数， GEODIST

- **georadius**

  `georadius<key><longitude><latitude>radius m|km|ft|mi`：以给定的经纬度为中心，找出某一半径内的元素，四个参数——经度、纬度、距离、单位

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/ac2625720536437d9fc4662caa6c7f42-61149c61ce4c30a23f39cb8e62fcfc2a-818760.png" alt="img" style="zoom:80%;" />

---

## 八、Jedis操作Redis

### 1.常用操作

**jedis所需的依赖**

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.3.0</version>
</dependency>
```

**连接Redis的注意事项**

- 开放端口（腾讯云直接控制台开发端口）

  ```
  firewall-cmd --permanent --add-port=6379/tcp
  firewall-cmd --permanent --query-port=6379/tcp # 查看
  firewall-cmd --permanent --remove-port=6379/tcp
  ```

- redis.conf 中注释掉 bind 127.0.0.1 ,然后 protected-mode no，设置完后[重启 Redis](#shutdown)

**测试程序**

```java
public static void main(String[] args) {
    //创建Jedis对象
    Jedis jedis = new Jedis("150.158.27.170", 6379);
    jedis.auth(password);
    //测试
    String ping = jedis.ping();
    System.out.println(ping); // PONG
}
```

**测试相关数据类型**

- 操作 Key 和 String

  ```java
  //操作key string
  @Test
  public void demo1() {
      Jedis jedis = getJedis();
      jedis.set("name","lucy");
  
      //获取
      String name = jedis.get("name");
      System.out.println(name);
  
      //设置多个key-value
      jedis.mset("k1","v1","k2","v2");
      List<String> mget = jedis.mget("k1", "k2");
      System.out.println(mget);
  
      // 获取所有的 keys
      Set<String> keys = jedis.keys("*");
      for(String key : keys) {
          System.out.println(key);
      }
      jedis.close();
  }
  ```

- 操作list

  ```java
  //操作list
  @Test
  public void demo2() {
      //创建Jedis对象
      Jedis jedis = getJedis();
      jedis.lpush("key1","lucy","mary","jack");
      List<String> values = jedis.lrange("key1", 0, -1);
      System.out.println(values); // [jack, mary, lucy]
      jedis.close();
  }
  ```

- 操作set

  ```java
  //操作set
  @Test
  public void demo3() {
      //创建Jedis对象
      Jedis jedis = getJedis();
      jedis.sadd("names","lucy");
      jedis.sadd("names","mary");
  
      Set<String> names = jedis.smembers("names");
      System.out.println(names); // [lucy, mary]
      jedis.close();
  }
  ```

- 操作hash

  ```java
  //操作hash
  @Test
  public void demo4() {
      //创建Jedis对象
      Jedis jedis = getJedis();
      jedis.hset("users","age","20");
      String hget = jedis.hget("users", "age");
      System.out.println(hget); // 20
      jedis.close();
  }
  ```

- 操作zset

  ```java
  //操作zset
  @Test
  public void demo5() {
      //创建Jedis对象
      Jedis jedis = new Jedis("192.168.242.110", 6379);
      jedis.zadd("china",100d,"shanghai");
      Set<String> china = jedis.zrange("china", 0, -1);
      System.out.println(china); // [shanghai]
      jedis.close();
  }
  ```

### 2.小实战(手机验证码)

- 输入手机号，点击发送后随机生成 6 位数字，2 分钟有效
- 输入验证码，点击验证，返回成功或失败
- 每个手机号每天只能输入 3 次

```java
public class PhoneCode {
    public static void main(String[] args) {
        //模拟验证码发送
        verifyCode("08061540919");

        //模拟验证码校验
        //getRedisCode("08061540919","744725");
    }

    /**
     * 生成六位数字验证码
     * @return
     */
    public static String getCode(){
        Random random = new Random();
        StringBuilder code = new StringBuilder();
        for(int i = 0; i < 6; i++){
            int index = random.nextInt(10);
            code.append(index);
        }
        System.out.println("生成验证码: " + code.toString());
        return code.toString();
    }

    /**
     * 每个手机每天只能发送三次，验证码放到 redis 中，设置过期时间 120
     * @param phone
     */
    public static void verifyCode(String phone) {
        //创建Jedis对象 连接Redis
        Jedis jedis = getJedis();

        //拼接key
        //手机发送次数key
        String countKey = "VerifyCode"+phone+":count";
        //验证码key
        String codeKey = "VerifyCode"+phone+":code";

        String count = jedis.get(countKey);
        if(count == null){
            //没有发送次数，第一次发送
            //设置发送次数是1
            jedis.setex(countKey,24*60*60,"1");
        }else if(Integer.parseInt(count) <= 2){
            //发送次数+1
            jedis.incr(countKey);
        }else{
            System.out.println("今天发送次数已经超过三次");
            jedis.close();
            return;
        }

        //发送验证码放到redis里面
        System.out.println("保存验证码");
        String vcode = getCode();
        jedis.setex(codeKey,120,vcode);
        jedis.close();

    }

    public static void getRedisCode(String phone,String code) {
        //从redis获取验证码
        Jedis jedis = getJedis();
        //验证码key
        String codeKey = "VerifyCode"+phone+":code";
        String redisCode = jedis.get(codeKey);
        //判断
        if(redisCode.equals(code)) {
            System.out.println("成功");
        }else {
            System.out.println("失败");
        }
        jedis.close();
    }
}
```

---

## 九、SpringBoot整合

> - springboot 2.x 后 ，原来默认使用的 Jedis 被 lettuce 替换，配置文件中的配置注意使用 lettuce
> - jedis：采用的直连，多个线程操作的话，是不安全的。如果要避免不安全，使用 jedis pool 连接池！更像 BIO 模式
> - lettuce：采用 netty，实例可以在多个线程中共享，不存在线程不安全的情况！可以减少线程数据了，更像 NIO 模式

### 1.快速测试

- 创建 springboot 项目，导入依赖

  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
  </dependency>
  ```

- 编写配置文件，连接 Redis

  ```yaml
  spring:
    redis:
      host: 150.158.27.170
      port: 6379 # 默认 6379
      database: 0 # 数据库索引，默认为 0
      connect-timeout: 100000 # 连接超时时间（毫秒）
      lettuce:
        pool:
          max-active: 20 # 连接池最大连接数，使用负值表示没有限制
          max-wait: -1 # 最大阻塞时间，负数表示没有限制
          max-idle: 5 # 连接池中最大空闲连接
          min-idle: 0 # 最小空闲连接
      password: #
  ```

- 测试方法

  ```java
  // 这就是之前 RedisAutoConfiguration 源码中的 Bean
  @Autowired
  private RedisTemplate redisTemplate;
  @Test
  void testRedis() {
      /** redisTemplate 操作不同的数据类型，API 和 Redis 中的是一样的
           * opsForValue 类似于 Redis 中的 String
           * opsForList 类似于 Redis 中的 List
           * opsForSet 类似于 Redis 中的 Set
           * opsForHash 类似于 Redis 中的 Hash
           * opsForZSet 类似于 Redis 中的 ZSet
           * opsForGeo 类似于 Redis 中的 Geospatial
           * opsForHyperLogLog 类似于 Redis 中的 HyperLogLog
       */
      // 除了基本的操作，常用的命令都可以直接通过 redisTemplate.xxxx 操作，比如事务、基本的CRUD……
  
      // 和数据库相关的操作都需要通过连接操作
      //获取Redis的连接对象
      //RedisConnection connection = redisTemplate.getConnectionFactory().getConnection();
      //connection.flushDb();
  
      redisTemplate.opsForValue().set("key", "key");
      System.out.println(redisTemplate.opsForValue().get("key"));
  }
  ```

### 2.序列化(自定义RedisTemplate模板)

**为什么要序列化？**

**测试一，使用 JSON 序列化 value**

- 新建User类，先不要序列化

  ```java
  @Component
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  public class User{
      private String name;
      private int age;
  }
  ```

- Redis 中存入 user 对象

  ```java
  @Test
  void testRedis() {
      // 使用 JSON 序列化
    	// 或者可以直接让 User 类继承 Serializable 接口，可以不用 mapper 序列化，此时使用 JDK 默认的序列化方式
      String user = new ObjectMapper().writeValueAsString(new User("zhao", 123));
      redisTemplate.opsForValue().set("user", user);
      System.out.println(redisTemplate.opsForValue().get("user"));
  }
  ```

- 结果

  - idea 控制台输出：User(name=zhao, age=123)
  - Linux 中 redis 控制台获取所有 key："\xac\xed\x00\x05t\x00\x04user"

**测试二，redis直接存储对象**（对象没有实现 Serializable 接口）

```java
@Test
public void test() {
    User user = new User("zhao", 123);
    redisTemplate.opsForValue().set("user",user);
    System.out.println(redisTemplate.opsForValue().get("user"));
}
```

测试结果报错

> 综上所述，需要序列化

==如何序列化==

- 首先对象要求序列化

  ```java
  @Component
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  public class User implements Serializable {
      private String name;
      private int age;
  }
  ```

- 自定义 RedisTemplate，在定义的 redisTemplate 设置序列化

  ```java
  @Configuration
  public class RedisConfig extends CachingConfigurerSupport {
      /**
       * 配置Jackson2JsonRedisSerializer序列化策略
       * */
      private Jackson2JsonRedisSerializer<Object> serializer() {
          // 使用Jackson2JsonRedisSerializer来序列化和反序列化redis的value值
          Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<>(Object.class);
          ObjectMapper objectMapper = new ObjectMapper();
  
          // 指定要序列化的域，field、get 和 set 及修饰符范围，ANY是都有包括 private 和 public
          objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
  
          // 不序列化 null 值
          objectMapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
  
          // 指定序列化输入的类型，类必须是非final修饰的，final修饰的类，比如 String, Integer 等会跑出异常，不指定则可以序列化 final 类型
          // 且不做任何验证，允许所有子类型
          // 不添加该语句输出则会{\"name\":\"zhao\",\"age\":123}
          objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, ObjectMapper.DefaultTyping.NON_FINAL);
  
          jackson2JsonRedisSerializer.setObjectMapper(objectMapper);
          return jackson2JsonRedisSerializer;
      }
  
      @Bean
      public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory){
          // 为了开发方便，直接使用<String, Object>
          RedisTemplate<String, Object> template = new RedisTemplate<>();
          // 用Jackson2JsonRedisSerializer来序列化和反序列化redis的value值
          template.setConnectionFactory(redisConnectionFactory);
          // String 的序列化
          StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();
          // key 采用 String 的序列化方式
          template.setKeySerializer(stringRedisSerializer);
          // Hash 的 key 采用 String 的序列化方式
          template.setHashKeySerializer(stringRedisSerializer);
          // value 采用 jackson 的序列化方式
          template.setValueSerializer(serializer());
          // Hash 的 value 采用 jackson 的序列化方式
          template.setHashValueSerializer(serializer());
          // 把所有的配置 set 进 template
          template.afterPropertiesSet();
          return template;
      }
  }
  ```
  
- 继续测试

  ```java
  @Test
  void testRedis() {
      redisTemplate.opsForValue().set("user", new User("zhao", 123));
      System.out.println(redisTemplate.opsForValue().get("user")); // User(name=zhao, age=123)
  }
  ```

  ```
  127.0.0.1:6379> keys *
  "user"
  127.0.0.1:6379> get user
  "[\"com.example.redis_springboot.entity.User\",{\"name\":\"zhao\",\"age\":123}]"
  ```

### 3.源码

```java
/**
* spring-boot-autoconfigure-2.6.2.jar -> spring.factories 中定义了所有需要加载的 autoconfiguration
**/
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({RedisOperations.class})
@EnableConfigurationProperties({RedisProperties.class})
@Import({LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class})
public class RedisAutoConfiguration {
    public RedisAutoConfiguration() {
    }

    @Bean
    @ConditionalOnMissingBean(name = {"redisTemplate"}) // 可以定制 redisTemplate
    @ConditionalOnSingleCandidate(RedisConnectionFactory.class)
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        // 默认的 redisTemplate 没有过多的设置，两个类型都是 object，可以定制 String 类型
        RedisTemplate<Object, Object> template = new RedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnSingleCandidate(RedisConnectionFactory.class)
    // String 是最常使用的类型，所以单独提出来了
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory) {
        return new StringRedisTemplate(redisConnectionFactory);
    }
}
```

### 4.Redis 工具类
<details>
<summary>工具类</summary>
@Component
public final class RedisUtil {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    // =============================common============================
    /**
     * 指定缓存失效时间
     * @param key  键
     * @param time 时间(秒)
     */
    public boolean expire(String key, long time) {
        try {
            if (time > 0) {
                redisTemplate.expire(key, time, TimeUnit.SECONDS);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
    /**
     * 根据key 获取过期时间
     * @param key 键 不能为null
     * @return 时间(秒) 返回0代表为永久有效
     */
    public long getExpire(String key) {
        return redisTemplate.getExpire(key, TimeUnit.SECONDS);
    }
    /**
     * 判断key是否存在
     * @param key 键
     * @return true 存在 false不存在
     */
    public boolean hasKey(String key) {
        try {
            return redisTemplate.hasKey(key);
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
    /**
     * 删除缓存
     * @param key 可以传一个值 或多个
     */
    @SuppressWarnings("unchecked")
    public void del(String... key) {
        if (key != null && key.length > 0) {
            if (key.length == 1) {
                redisTemplate.delete(key[0]);
            } else {
                redisTemplate.delete((Collection<String>) CollectionUtils.arrayToList(key));
            }
        }
    }
    // ============================String=============================
    /**
     * 普通缓存获取
     * @param key 键
     * @return 值
     */
    public Object get(String key) {
        return key == null ? null : redisTemplate.opsForValue().get(key);
    }
    /**
     * 普通缓存放入
     * @param key   键
     * @param value 值
     * @return true成功 false失败
     */
    public boolean set(String key, Object value) {
        try {
            redisTemplate.opsForValue().set(key, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
    /**
     * 普通缓存放入并设置时间
     * @param key   键
     * @param value 值
     * @param time  时间(秒) time要大于0 如果time小于等于0 将设置无限期
     * @return true成功 false 失败
     */
    public boolean set(String key, Object value, long time) {
        try {
            if (time > 0) {
                redisTemplate.opsForValue().set(key, value, time, TimeUnit.SECONDS);
            } else {
                set(key, value);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
    /**
     * 递增
     * @param key   键
     * @param delta 要增加几(大于0)
     */
    public long incr(String key, long delta) {
        if (delta < 0) {
            throw new RuntimeException("递增因子必须大于0");
        }
        return redisTemplate.opsForValue().increment(key, delta);
    }
    /**
     * 递减
     * @param key   键
     * @param delta 要减少几(小于0)
     */
    public long decr(String key, long delta) {
        if (delta < 0) {
            throw new RuntimeException("递减因子必须大于0");
        }
        return redisTemplate.opsForValue().increment(key, -delta);
    }
    // ================================Map=================================
    /**
     * HashGet
     * @param key  键 不能为null
     * @param item 项 不能为null
     */
    public Object hget(String key, String item) {
        return redisTemplate.opsForHash().get(key, item);
    }
    /**
     * 获取hashKey对应的所有键值
     * @param key 键
     * @return 对应的多个键值
     */
    public Map<Object, Object> hmget(String key) {
        return redisTemplate.opsForHash().entries(key);
    }
    /**
     * HashSet
     * @param key 键
     * @param map 对应多个键值
     */
    public boolean hmset(String key, Map<String, Object> map) {
        try {
            redisTemplate.opsForHash().putAll(key, map);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
    /**
     * HashSet 并设置时间
     * @param key  键
     * @param map  对应多个键值
     * @param time 时间(秒)
     * @return true成功 false失败
     */
    public boolean hmset(String key, Map<String, Object> map, long time) {
        try {
            redisTemplate.opsForHash().putAll(key, map);
            if (time > 0) {
                expire(key, time);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
    /**
     * 向一张hash表中放入数据,如果不存在将创建
     *
     * @param key   键
     * @param item  项
     * @param value 值
     * @return true 成功 false失败
     */
    public boolean hset(String key, String item, Object value) {
        try {
            redisTemplate.opsForHash().put(key, item, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
    /**
     * 向一张hash表中放入数据,如果不存在将创建
     *
     * @param key   键
     * @param item  项
     * @param value 值
     * @param time  时间(秒) 注意:如果已存在的hash表有时间,这里将会替换原有的时间
     * @return true 成功 false失败
     */
    public boolean hset(String key, String item, Object value, long time) {
        try {
            redisTemplate.opsForHash().put(key, item, value);
            if (time > 0) {
                expire(key, time);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
    /**
     * 删除hash表中的值
     *
     * @param key  键 不能为null
     * @param item 项 可以使多个 不能为null
     */
    public void hdel(String key, Object... item) {
        redisTemplate.opsForHash().delete(key, item);
    }
    /**
     * 判断hash表中是否有该项的值
     *
     * @param key  键 不能为null
     * @param item 项 不能为null
     * @return true 存在 false不存在
     */
    public boolean hHasKey(String key, String item) {
        return redisTemplate.opsForHash().hasKey(key, item);
    }
    /**
     * hash递增 如果不存在,就会创建一个 并把新增后的值返回
     *
     * @param key  键
     * @param item 项
     * @param by   要增加几(大于0)
     */
    public double hincr(String key, String item, double by) {
        return redisTemplate.opsForHash().increment(key, item, by);
    }
    /**
     * hash递减
     *
     * @param key  键
     * @param item 项
     * @param by   要减少记(小于0)
     */
    public double hdecr(String key, String item, double by) {
        return redisTemplate.opsForHash().increment(key, item, -by);
    }
    // ============================set=============================
    /**
     * 根据key获取Set中的所有值
     * @param key 键
     */
    public Set<Object> sGet(String key) {
        try {
            return redisTemplate.opsForSet().members(key);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
    /**
     * 根据value从一个set中查询,是否存在
     *
     * @param key   键
     * @param value 值
     * @return true 存在 false不存在
     */
    public boolean sHasKey(String key, Object value) {
        try {
            return redisTemplate.opsForSet().isMember(key, value);
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
    /**
     * 将数据放入set缓存
     *
     * @param key    键
     * @param values 值 可以是多个
     * @return 成功个数
     */
    public long sSet(String key, Object... values) {
        try {
            return redisTemplate.opsForSet().add(key, values);
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }
    /**
     * 将set数据放入缓存
     *
     * @param key    键
     * @param time   时间(秒)
     * @param values 值 可以是多个
     * @return 成功个数
     */
    public long sSetAndTime(String key, long time, Object... values) {
        try {
            Long count = redisTemplate.opsForSet().add(key, values);
            if (time > 0)
                expire(key, time);
            return count;
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }
    /**
     * 获取set缓存的长度
     *
     * @param key 键
     */
    public long sGetSetSize(String key) {
        try {
            return redisTemplate.opsForSet().size(key);
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }
    /**
     * 移除值为value的
     *
     * @param key    键
     * @param values 值 可以是多个
     * @return 移除的个数
     */
    public long setRemove(String key, Object... values) {
        try {
            Long count = redisTemplate.opsForSet().remove(key, values);
            return count;
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }
    // ===============================list=================================
    /**
     * 获取list缓存的内容
     *
     * @param key   键
     * @param start 开始
     * @param end   结束 0 到 -1代表所有值
     */
    public List<Object> lGet(String key, long start, long end) {
        try {
            return redisTemplate.opsForList().range(key, start, end);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
    /**
     * 获取list缓存的长度
     *
     * @param key 键
     */
    public long lGetListSize(String key) {
        try {
            return redisTemplate.opsForList().size(key);
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }
    /**
     * 通过索引 获取list中的值
     *
     * @param key   键
     * @param index 索引 index>=0时， 0 表头，1 第二个元素，依次类推；index<0时，-1，表尾，-2倒数第二个元素，依次类推
     */
    public Object lGetIndex(String key, long index) {
        try {
            return redisTemplate.opsForList().index(key, index);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
    /**
     * 将list放入缓存
     *
     * @param key   键
     * @param value 值
     */
    public boolean lSet(String key, Object value) {
        try {
            redisTemplate.opsForList().rightPush(key, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
    /**
     * 将list放入缓存
     * @param key   键
     * @param value 值
     * @param time  时间(秒)
     */
    public boolean lSet(String key, Object value, long time) {
        try {
            redisTemplate.opsForList().rightPush(key, value);
            if (time > 0)
                expire(key, time);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
    /**
     * 将list放入缓存
     *
     * @param key   键
     * @param value 值
     * @return
     */
    public boolean lSet(String key, List<Object> value) {
        try {
            redisTemplate.opsForList().rightPushAll(key, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
    /**
     * 将list放入缓存
     *
     * @param key   键
     * @param value 值
     * @param time  时间(秒)
     * @return
     */
    public boolean lSet(String key, List<Object> value, long time) {
        try {
            redisTemplate.opsForList().rightPushAll(key, value);
            if (time > 0)
                expire(key, time);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
    /**
     * 根据索引修改list中的某条数据
     *
     * @param key   键
     * @param index 索引
     * @param value 值
     * @return
     */
    public boolean lUpdateIndex(String key, long index, Object value) {
        try {
            redisTemplate.opsForList().set(key, index, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
    /**
     * 移除N个值为value
     *
     * @param key   键
     * @param count 移除多少个
     * @param value 值
     * @return 移除的个数
     */
    public long lRemove(String key, long count, Object value) {
        try {
            Long remove = redisTemplate.opsForList().remove(key, count, value);
            return remove;
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }
}

---

## 十、Redis事务操作

### 1.事务的概述

- Redis 事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断

- Redis 事务的主要作用就是**串联多个命令防止别的命令插队**

### 2.Multi、Exec、discard

- 从输入 Multi 命令开始，输入的命令都会依次进入命令队列中，但不会执行，直到输入 Exec 后，Redis 会将之前的命令队列中的命令 **依次执行**

- 组队的过程中可以通过 discard

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,size_16,color_FFFFFF,t_270-fe4a48b969d2af7f36b71cc635f09ca7-fdb997" alt="img" style="zoom:80%;" />

**实例**

组队成功，提交成功

```
127.0.0.1:6379(TX)> set key 1
QUEUED
127.0.0.1:6379(TX)> get key
QUEUED
127.0.0.1:6379(TX)> exec
1) OK
2) "1"
```

组队阶段报错，提交失败

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379(TX)> set k1 1
QUEUED
127.0.0.1:6379(TX)> set k2 2
QUEUED
127.0.0.1:6379(TX)> discard
OK
```

组队成功，提交有成功有失败情况

### 3.事务的错误处理

- 组队时某个命令出现了错误，执行时整个的所有队列都会被取消

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,size_16,color_FFFFFF,t_470-ae1b01477afc0cd52731dcbc8ea7951b-3289f9" alt="img" style="zoom:80%;" />

  ```
  127.0.0.1:6379> set k1 1
  OK
  127.0.0.1:6379> set k2 2
  OK
  127.0.0.1:6379> set k3
  (error) ERR wrong number of arguments for 'set' command
  127.0.0.1:6379> exec
  (error) ERR EXEC without MULTI
  ```

- 如果执行阶段某个命令出现了错误，则只有报错的命令不会被执行，而其他的命令都会执行，不会回滚

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,size_16,color_FFFFFF,t5_70-8574c623746bb68f783b92a589221b0e-3fee12" alt="img" style="zoom:80%;" />
  
  ```
  127.0.0.1:6379> multi
  OK
  127.0.0.1:6379(TX)> set k1 v1
  QUEUED
  127.0.0.1:6379(TX)> incr k1
  QUEUED
  127.0.0.1:6379(TX)> set k2 v2
  QUEUED
  127.0.0.1:6379(TX)> exec
  1) OK
  2) (error) ERR value is not an integer or out of range
  3) OK
  ```

### 4.事务冲突问题

#### 4.1 实例

一个请求想给金额减 8000

一个请求想给金额减 5000

一个请求想给金额减 1000

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/178a125a4a11408a9cf43ed1f63dee14-cc8fa4e5bb0bc903a76a4c55c6de36f0-7158e0.png" alt="img" style="zoom:80%;" />

#### 4.2 悲观锁

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/148464141b62431ab1b5f3d1d98acd69-4aea3cf54ae127a6786bbd971748d301-c34f98.png" alt="img" style="zoom:80%;" />

> ​    悲观锁（Pessimistic Lock），顾名思义，就是很悲观，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会block直到它拿到锁。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。

#### 4.3 乐观锁

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/5358bb00c39e4c538d5bc4a668695375-23a90b71477c7249369b3d3397d31111-71acb5.png" alt="img" style="zoom:80%;" />

> ​    乐观锁（Optimistic Lock），顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号等机制。乐观锁适用于多读的应用类型，这样可以提高吞吐量。Redis 就是利用这种 check-and-set 机制实现事务的。

#### 4.4 `WATCH <key><key>…`

> 在执行 multi 之前，先执行 watch key1 [key2]，可以监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。

演示

```
127.0.0.1:6379> watch balance
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379(TX)> incrby balance 10
QUEUED
127.0.0.1:6379(TX)> exec
1) (integer) 110
----------------------------------------
#与此同时
127.0.0.1:6379> watch balance
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379(TX)> incrby balance 20
QUEUED
127.0.0.1:6379(TX)> exec
(nil) # 失败
```

> 在开启事务的时候，在执行前先修改一下信息，就会执行失败，这是 watch key 的作用

#### 4.5 unwatch

- 取消 WATCH 命令对所有 key 的监视
- 如果在执行 WATCH 命令之后，EXEC 命令或DISCARD 命令先被执行了的话，那么就不需要再执行UNWATCH了

#### 4.6 Redis 事务三特性

- 单独的隔离操作
  - 事务中的所有命令都会序列化，按顺序的执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断
- 没有隔离级别的概念
  - 队列中的命令没有提交前都不会实际被执行
- 不保证原子性
  - 事务中如果有一条命令执行失败，其后的命令仍然会被执行，没有回滚

### 5.Redis事务-秒杀案例

#### 5.1 环境搭建

- 创建 springBoot 项目，添加 webapp/WEB-INF 目录

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220118020316118-b2414d76668d98a36f070a03813d3159-c594e0.png" alt="image-20220118020316118" style="zoom:80%;" />

- 导入 jar 包

  ```xml
  <!-- 引入SpringBoot内嵌Tomcat对jsp的解析依赖，不添加这个解析不了jsp -->
  <dependency>
      <groupId>org.apache.tomcat.embed</groupId>
      <artifactId>tomcat-embed-jasper</artifactId>
  </dependency>
  <!-- JSTL -->
  <dependency>
      <groupId>javax.servlet</groupId>
      <artifactId>jstl</artifactId>
  </dependency>
  ```

- 创建 index.jsp

  ```jsp
  <%@ page language="java" contentType="text/html; charset=UTF-8"
           pageEncoding="UTF-8"%>
  <!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
  <html>
  <head>
      <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
      <title>Insert title here</title>
  </head>
  <body>
  <h1>iPhone 13 Pro !!!  1元秒杀！！！</h1>
  
  <p>${pageContext.request.contextPath}</p>
  <form id="msform" action="${pageContext.request.contextPath}/doseckill" enctype="application/x-www-form-urlencoded">
      <input type="hidden" id="prodid" name="prodid" value="0101">
      <input type="button"  id="miaosha_btn" name="seckill_btn" value="秒杀点我"/>
  </form>
  
  </body>
  <script  type="text/javascript" src="https://ajax.aspnetcdn.com/ajax/jquery/jquery-3.1.1.min.js"></script>
  <script  type="text/javascript">
      $(function(){
          $("#miaosha_btn").click(function(){
              var url=$("#msform").attr("action");
              $.post(url,$("#msform").serialize(),function(data){
                  if(data=="false"){
                      alert("抢光了" );
                      $("#miaosha_btn").attr("disabled",true);
                  }
              } );
          })
      })
  </script>
  </html>
  ```
  
- 配置 InternalResourceViewResolver

  ```yaml
  spring:
    mvc:
      view:
        prefix: /WEB-INF/
        suffix: .jsp
  ```
  
- IndexController

  ```java
  @Controller
  public class IndexController {
      @GetMapping("/")
      String index() {
          // 会被 InternalResourceViewResolver 解析
          return "index";
      }
  }
  ```
  
- SecKill_redis

  ```java
  @Controller
  public class DoSecKillController {
  
      @Autowired
      RedisTemplate<String, Object> redisTemplate;
  
      @PostMapping("/doseckill")
      public void doSecKill(String prodid, HttpServletResponse response) throws IOException {
          String userId = new Random().nextInt(50000) + "";
          response.getWriter().print(doSecKill(userId, prodid));
      }
  
      private boolean doSecKill(String userId, String proId) {
          if (!StringUtils.hasText(userId) || !StringUtils.hasText(proId)) {
              return false;
          }
  
          // 库存 key
          String stockKey = proId + ":stock";
          // 秒杀成功用户 key
          String userKey = proId + ":user";
  
          // 获取不到 stockKey 表示还未开始
          Integer stock = (Integer) redisTemplate.opsForValue().get(stockKey);
          if (Objects.isNull(stock)) {
              System.out.println("not yet start");
              return false;
          }
  
          // 库存小于等于 0 说明已经结束
          if (stock <= 0) {
              System.out.println("kill campaign has finished");
              return false;
          }
  
          // 用户已经秒杀成功，不能再次秒杀
          Boolean userHasKilled = redisTemplate.opsForSet().isMember(userKey, userId);
          if (userHasKilled) {
              System.out.println("has already killed successfully, can't kill again");
              return false;
          }
  
          // 添加秒杀用户
          redisTemplate.opsForValue().decrement(stockKey);
          redisTemplate.opsForSet().add(userKey, userId);
          System.out.println("killed successfully");
          return true;
      }
      
  }
  ```
  
- 测试

  > 先在 redis 中设置库存数10，启动项目，进项秒杀

#### 5.2 并发暴露出来的问题

> 使用ab工具（或者Jmeter）模拟
>
> CentOS6 默认安装
>
> CentOS7 需要手动安装

1. ab

   - 使用命令安装 ab

     ```bash
     yum -y install httpd-tools
     ```

   - 编写参数文件

     ```bash
     vim postfile 模拟表单提交参数，以&符号结尾，存放在 ~（home/用户名）目录下
     内容：prodid=0101&
     ```

   - 启动项目，通过ab命令并发秒杀

     ```bash
     ab -n 2000 -c 200 -k -p ~/postfile -T application/x-www-form-urlencoded http://192.168.0.43:8080/doseckill
     -n：次数
     -c：其中并发次数
     -p：post 提交
     ~/postfile：参数文件
     -T 如果是 post/put 需要设置为 application/x-www-form-urlencoded 
     ```

2. Jmeter

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220118025612786-b2a24df2db75d07d205d8b597154cf45-41d2b6.png" alt="image-20220118025612786" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220118025559949-d95d8216bf4e5ef302dfce0ab816d0ed-7a12b6.png" alt="image-20220118025559949" style="zoom:80%;" />

- 并发暴露出来的问题

  - 会出现超卖问题，卖光了还能秒杀成功，库存为负数

    - 超卖产生的原因

    <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,size_16,color_FFFFFF,t_750-b752efb862187d0dc7860544c4433b1e-865821" alt="img" style="zoom:80%;" />

  - 连接超时问题

  - 商品遗留问题：秒杀结束了，还有商品库存

#### 5.3 问题的解决

**连接超时问题（利用连接池）**

- 创建工具类

  ```java
  public class JedisPoolUtil {
      private static volatile JedisPool jedisPool = null;
  
      private JedisPoolUtil() {
      }
  
      public static JedisPool getJedisPoolInstance() {
          if (null == jedisPool) {
              synchronized (JedisPoolUtil.class) {
                  if (null == jedisPool) {
                      JedisPoolConfig poolConfig = new JedisPoolConfig();
                      poolConfig.setMaxTotal(200);
                      poolConfig.setMaxIdle(32);
                      poolConfig.setMaxWaitMillis(100*1000);
                      poolConfig.setBlockWhenExhausted(true);
                      poolConfig.setTestOnBorrow(true);  // ping  PONG
  
                      jedisPool = new JedisPool(poolConfig, "192.168.242.110", 6379, 60000);
                  }
              }
          }
          return jedisPool;
      }
  
      public static void release(JedisPool jedisPool, Jedis jedis) {
          if (null != jedis) {
              jedisPool.returnResource(jedis);
          }
      }
  }
  ```

  ```java
  //2 连接redis
  //Jedis jedis =new Jedis("192.168.242.110",6379);
  
  //通过连接池获取连接 redis 的对象
  JedisPool jedisPoolInstance = JedisPoolUtil.getJedisPoolInstance();
  Jedis jedis = jedisPoolInstance.getResource();
  ```

**利用乐观锁淘汰用户，解决超卖问题**

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,size_16,color_FFFFFF,t_706-185510c6527aa135d5fc104d5fa55c66-f5a534" alt="img" style="zoom:80%;" />

```java
// RedisTemplate 的事务操作
private boolean doSecKill(String userId, String proId) {
    if (!StringUtils.hasText(userId) || !StringUtils.hasText(proId)) {
        return false;
    }

    // 库存 key
    String stockKey = proId + ":stock";
    // 秒杀成功用户 key
    String userKey = proId + ":user";

    // 使用 SessionCallback 实现事务
    SessionCallback<Boolean> callback = new SessionCallback<Boolean>() {
        @Override
        @SuppressWarnings("unchecked")
        public Boolean execute(RedisOperations ops) throws DataAccessException {
            // 用户已经秒杀成功，不能再次秒杀
            Boolean userHasKilled = ops.opsForSet().isMember(userKey, userId);
            if (userHasKilled) {
                System.out.println("has already killed successfully, can't kill again");
                return false;
            }
            if (!ops.hasKey(stockKey)) {
                System.out.println("kill campaign is not started");
                return false;
            }
            // watch key
            ops.watch(stockKey);
            // 取出 key
            Integer stockNum = (Integer) ops.opsForValue().get(stockKey);
            // 小于 0 则表示货没了
            if (stockNum <= 0) {
                return false;
            }
            // 开启事务
            ops.multi();
            // Write operations
            ops.opsForValue().decrement(stockKey);
            ops.opsForSet().add(userKey, userId);
            // 执行
            if (ops.exec().size() > 0) {
                return true;
            }
            return false;
        }
    };

    if (redisTemplate.execute(callback)) {
        System.out.println("killed successfully");
        return true;
    }

    System.out.println("kill failed");
    return false;
}
```

```java
// Jedis 的事务操作
public static boolean doSecKill(String uid,String prodid) throws IOException {
    //1 uid和prodid非空判断
    if(uid == null || prodid == null){
        return false;
    }

    //2 连接redis
    //Jedis jedis =new Jedis("192.168.242.110",6379);

    //通过连接池获取连接redis的对象
    JedisPool jedisPoolInstance = JedisPoolUtil.getJedisPoolInstance();
    Jedis jedis = jedisPoolInstance.getResource();

    //3 拼接key
    // 3.1 库存key
    String kcKey = "sk:"+prodid+":qt";
    // 3.2 秒杀成功用户key
    String userKey = "sk:"+prodid+":user";

    //监视库存
    jedis.watch(kcKey);

    //4 获取库存，如果库存null，秒杀还没有开始
    String kc = jedis.get(kcKey);
    if(kc == null){
        System.out.println("秒杀还没开始，请稍等");
        jedis.close();
        return false;
    }

    // 5 判断用户是否重复秒杀操作
    if(jedis.sismember(userKey, uid)){
        System.out.println("每个用户只能秒杀成功一次，请下次再来");
        jedis.close();
        return false;
    }

    //6 判断如果商品数量，库存数量小于1，秒杀结束
    if(Integer.parseInt(kc) < 1){
        System.out.println("秒杀结束，请下次参与");
        jedis.close();
        return false;
    }

    //7 秒杀过程
    //使用事务
    Transaction multi = jedis.multi();

    //组队操作
    multi.decr(kcKey);
    multi.sadd(userKey,uid);

    //执行
    List<Object> results = multi.exec();

    if(results == null || results.size()==0) {
        System.out.println("秒杀失败了....");
        jedis.close();
        return false;
    }

    System.out.println("用户" + uid + "秒杀成功");
    jedis.close();
    return true;
}
```

##### ==商品遗留问题（Lua）==

> 用 Lua 脚本解决商品遗留问题，相当于实现了悲观锁
>
> - Lua 是一个小巧的脚本语言，Lua 脚本可以很容易的被 C/C++ 代码调用，也可以反过来调用 C/C++ 的函数，Lua 并没有提供强大的库，一个完整的 Lua 解释器不过 200k，所以 Lua 不适合作为开发独立应用程序的语言，而是作为嵌入式脚本语言。很多应用程序、游戏使用 lua 作为自己的嵌入式脚本语言，以此来实现可配置性、可扩展性
>
> - 将复杂的或者多步的 redis 操作，写为一个脚本，一次提交给 redis 执行，减少反复连接 redis 的次数，提升性能
>
> - lua 脚本是类似 redis 事务，有一定的原子性，不会被其他命令插队，可以完成一些 redis 事务性的操作
>
> - 但是注意 redis 的 lua 脚本功能，只有在 Redis 2.6 以上的版本才可以使用
>
> - 利用 lua 脚本淘汰用户，解决超卖问题
>
> - redis 2.6 版本以后，通过 lua 脚本解决争抢问题，实际上是 redis 利用其单线程的特性，用任务队列的方式解决多任务并发问题
>
> <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,size_16,color_FFFFFF,t_7066-9e99d50c60dfd745ff92ee1cc44b4e62-d0ddb1" alt="img" style="zoom:80%;" />

```java
private boolean doSecKill(String userId, String proId) {
    if (!StringUtils.hasText(userId) || !StringUtils.hasText(proId)) {
        return false;
    }

    String lua = "local userid = KEYS[1];\n" +
        "local proId = KEYS[2];\n" +
        "local stockKey = proId .. ':stock';\n" +
        "local userKey = proId .. ':user';\n" +
        "if tonumber(redis.call(\"exists\", stockKey))==0 then\n" +
        "    return -1\n" +
        "end;\n" +
        "local userExists = redis.call('sismember',userKey,userid);\n" +
        "if tonumber(userExists) == 1 then \n" +
        "    return 2\n" +
        "end;\n" +
        "local num = redis.call('get' ,stockKey);\n" +
        "if tonumber(num) <= 0 then\n" +
        "    return 0\n" +
        "else\n" +
        "    redis.call(\"decr\", stockKey);\n" +
        "    redis.call(\"sadd\", userKey, userid);\n" +
        "end\n" +
        "return 1;";

    // 不建议这样写
    Long result = redisTemplate.execute((RedisConnection connection) -> connection.eval(
        lua.getBytes(), // 脚本
        ReturnType.INTEGER, // 返回值类型
        2, // KEY 参数数量
        userId.getBytes(), // KEY1 参数
        proId.getBytes() // KEY2 参数
    ));
    if (result.equals(0L)) {
        System.err.println("已抢空！！");
    }else if(result == 1) {
        System.out.println("抢购成功！！！！");
        return true;
    }else if(result == 2) {
        System.err.println("该用户已抢过！！");
    }else{
        System.err.println("抢购异常！！");
    }
    return false;
}
```

[优化之LUA脚本保证删除的原子性](#优化之LUA脚本保证删除的原子性)

```lua
-- lua 脚本
local userid = KEYS[1];
local proId = KEYS[2];
local stockKey = proId .. ':stock';
local userKey = proId .. ':user';
if tonumber(redis.call("exists", stockKey))==0 then
    return -1
end;
local userExists = redis.call('sismember',userKey,userid);
if tonumber(userExists) == 1 then 
    return 2
end;
local num = redis.call('get' ,stockKey);
if tonumber(num) <= 0 then
    return 0
else
    redis.call("decr", stockKey);
    redis.call("sadd", userKey, userid);
end
return 1;
```

---

## 十一、Redis持久化

### 1.RDB持久化

> Redis DataBase：在指定的时间间隔内将内存中的数据集`快照`写入磁盘， 也就是行话讲的 Snapshot 快照，它恢复时是将快照文件直接读到内存里

#### 1.1 持久化执行流程

> ​    Redis 会单独创建（fork）一个子进程来进行持久化，会先将数据写入到一个临时文件中，待持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。 整个过程中，主进程是不进行任何 IO 操作的，这就确保了极高的性能如果需要进行大规模数据的恢复，且对于数据恢复的完整性不是非常敏感，那 RDB 方式要比 AOF 方式更加的高效。RDB 的缺点是最后一次持久化后的数据可能丢失。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,size_16,color_FFFFFF,t_708-7ea5c6238ab3d7ba0a8506008abddc9c-6f36b7" alt="img" style="zoom:80%;" />

#### 1.2 Fork过程概述

> - Fork 的作用是复制一个与当前进程一样的进程。新进程的所有数据（变量、环境变量、程序计数器等） 数值都和原进程一致，但是是一个全新的进程，并作为原进程的子进程
> - 在 Linux 程序中，fork() 会产生一个和父进程完全相同的子进程，但子进程在此后多会 exec 系统调用，出于效率考虑，Linux 中引入了“写时复制技术”
> - 一般情况父进程和子进程会共用同一段物理内存，只有进程空间的各段的内容要发生变化时，才会将父进程的内容复制一份给子进程

#### 1.3 相关命令

- save VS bgsave

  > save ：执行同步快照。手动保存。不建议
  >
  > **bgsave**：Redis 会在后台`异步`进行快照操作， **快照同时还可以响应客户端请求**
  >
  > lastsave：可以获取最后一次成功执行快照的时间

- fulshall 也会产生 dump.rbg 文件，不过是空的，没有意义

- save

  > 格式：save 秒钟 写操作次数
  >
  > RDB 是整个内存的压缩过的 Snapshot，RDB 的数据结构，可以配置复合的快照触发条件，默认是1分钟内改了1万次，或5分钟内改了100次，或15分钟内改了1次
  >
  > 禁用: 不设置 save 指令，或者给 save 传入空字符串

  ```bash
  # Save the DB to disk.
  #
  # save <seconds> <changes>
  #
  # Redis will save the DB if both the given number of seconds and the given
  # number of write operations against the DB occurred.
  #
  # Snapshotting can be completely disabled with a single empty string argument
  # as in following example:
  #
  # save ""
  #
  # Unless specified otherwise, by default Redis will save the DB:
  #   * After 3600 seconds (an hour) if at least 1 key changed
  #   * After 300 seconds (5 minutes) if at least 100 keys changed
  #   * After 60 seconds if at least 10000 keys changed
  #
  # You can set these explicitly by uncommenting the three following lines.
  #
  # save 3600 1
  # save 300 100
  # save 60 10000
  ```

#### 1.4 相关配置

- 在 redis.conf 中配置文件名称，默认为 dump.rdb

  ```bash
  # The filename where to dump the DB
  dbfilename dump.rdb
  ```

- 配置文件的位置，默认是启动位置

  ```bash
  # The working directory.
  #
  # The DB will be written inside this directory, with the filename specified
  # above using the 'dbfilename' configuration directive.
  #
  # The Append Only File will also be created inside this directory.
  #
  # Note that you must specify a directory here, not a file name.
  dir ./
  ```

  ```bash
  root@VM-16-13-ubuntu:/usr/local/bin# ll
  total 26620
  drwxr-xr-x  2 root root     4096 Jan 15 19:53 ./
  drwxr-xr-x 12 root root     4096 Nov 28 14:42 ../
  -rw-r--r--  1 root root      182 Jan 15 19:53 dump.rdb
  ```

- stop-writes-on-bgsave-error

  > 当 Redis 无法写入磁盘的话，直接关掉 Redis 的写操作，推荐 yes

- rdbcompression 压缩文件

  > 对于存储到磁盘中的快照，可以设置是否进行压缩存储。如果是的话，redis 会采用 LZF 算法进行压缩
  >
  > 如果你不想消耗 CPU 来进行压缩的话，可以设置为关闭此功能，推荐 yes

- rdbchecksum 检查完整性

  > 在存储快照后，还可以让 redis 使用 CRC64 算法来进行数据校验，但是这样做会增加大约 10% 的性能消耗，如果希望获取到最大的性能提升，可以关闭此功能->推荐 yes

- rdb 的备份

  > 先通过 config get dir 查询 rdb 文件的目录
  >
  > 将 *.rdb 的文件拷贝到别的地方 cp dump.rdb dump2.rdb
  >
  > **rdb的恢复**
  >
  > - 关闭 Redis
  > - 先把备份的文件拷贝到工作目录下 cp dump2.rdb dump.rdb
  > - 启动 Redis，备份数据会直接加载

#### 1.5 优势与劣势

**优势**

- 适合大规模的数据恢复
- 对数据完整性和一致性要求不高更适合使用
- 节省磁盘空间
- 恢复速度快

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,size_16,color_FFFFFF,t_70123-a4119051a4f82a954f153e99029641b3-e15140" alt="img" style="zoom:80%;" />

**劣势**

- Fork 的时候，内存中的数据被克隆了一份，大致2倍的膨胀性需要考虑
- 虽然 Redis 在 fork 时使用了 **写时拷贝技术**，但是如果数据庞大时还是比较消耗性能
- 在备份周期在一定间隔时间做一次备份，所以如果 Redis 意外 down 掉的话，就会丢失最后一次快照后的所有修改

#### 1.6 如何停止

> 动态停止 RDB：redis-cli config set save "" 
>
> save 后给空值，表示禁用保存策略

#### 1.7 总结

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,23-b0df23fa27143972b5d47cd41d667b7e-cc51d7" alt="img" style="zoom:80%;" />

> 注意持久化文件是在启动目录生成，不是一定在 /usr/local/bin 目录下

### 2 AOF持久化

#### 2.1 概述

> Append Only File：以**日志**的形式来记录每个写操作（增量保存），将 Redis 执行过的所有写指令记录下来（**读操作不记录**）， **只许追加文件但不可以改写文件**，redis 启动之初会读取该文件重新构建数据，换言之，redis 重启的话就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作

#### 2.2 AOF持久化流程

- 客户端的请求写命令会被追加到 AOF 缓冲区内

- AOF 缓冲区根据 AOF 持久化策略  [always,everysec,no] 将操作同步到磁盘的 AOF 文件中

- AOF 文件大小超过重写策略或手动重写时，会对 AOF 文件 rewrite 重写，压缩 AOF 文件容量

- Redis 服务重启时，会重新 load 加载 AOF 文件中的写操作达到数据恢复的目的

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/012496cf23cc497e9acdba1ca80a4315-9579153b7471e6808f2154582ddca0d4-b1fa75.png" alt="img" style="zoom:80%;" />

#### 2.3 AOF的开启与说明

- **默认不开启**

  > 开启：appendonly yes
  >
  > 可以在 redis.conf 中配置文件名称，默认为 appendonly.aof
  >
  > AOF 文件的保存路径，同 RDB 的路径一致

- **AOF和RDB同时开启，redis听谁的**

  > AOF 和 RDB 同时开启，系统默认取 AOF 的数据（数据不会存在丢失）

- **AOF启动/修复/恢复**

  > - AOF 的备份机制和性能虽然和 RDB 不同, 但是备份和恢复的操作同 RDB 一样，都是拷贝备份文件，需要恢复时再拷贝到 Redis 工作目录下，启动系统即加载。
  > - 正常恢复
  >   - 修改默认的  appendonly no，改为 yes
  >   - 将有数据的  aof 文件复制一份保存到对应目录(查看目录：config get dir)
  >   - 恢复：重启 redis 然后重新加载
  >
  > - 异常恢复
  >   - 修改默认的 appendonly no，改为 yes
  >   - 如遇到 AOF 文件损坏，通过 /usr/local/bin/redis-check-aof--fix appendonly.aof 进行恢复
  >   - 备份被写坏的 AOF 文件
  >   - 恢复：重启 redis，然后重新加载

- **AOF同步频率设置**

  > **appendfsync always**
  >
  >  始终同步，每次 Redis 的写入都会立刻记入日志；性能较差但数据完整性比较好
  >
  > **appendfsync everysec**
  >
  >  每秒同步，每秒记入日志一次，如果宕机，本秒的数据可能丢失
  >
  > **appendfsync no**
  >
  >  redis 不主动进行同步，把同步时机交给操作系统

- **Rewrite压缩**

  AOF 采用文件追加方式，文件会越来越大，为避免出现此种情况，新增了重写机制，当 AOF 文件的大小超过所设定的阈值时，Redis 就会启动 AOF 文件的内容压缩，只保留可以恢复数据的最小指令集，2.4 版本后自动触发，可以使用命令`bgrewriteaof`手动触发。

  - **重写原理，如何实现重写**

    > AOF文件持续增长而过大时，会 fork 出一条新进程来将文件重写(也是先写临时文件最后再 rename)，redis4.0 版本后的重写，是指上就是把 rdb 的快照，以二级制的形式附在新的aof 头部，作为已有的历史数据，替换掉原来的流水账操作。
    >
    > **no-appendfsync-on-rewrite：**
    >
    > - 如果 no-appendfsync-on-rewrite=yes，不写入 aof 文件只写入缓存，用户请求不会阻塞，但是在这段时间如果宕机会丢失这段时间的缓存数据。（降低数据安全性，提高性能）
    >
    > - 如果 no-appendfsync-on-rewrite=no，还是会把数据往磁盘里刷，但是遇到重写操作，可能会发生阻塞。（数据安全，但是性能降低）
    >
    > **触发机制，何时重写**
    >
    > - Redis 会记录上次重写时的 AOF 大小，默认配置是当 AOF 文件大小是上次 rewrite 后大小的一倍且文件大于 64M 时触发
    >
    > - 重写虽然可以节约大量磁盘空间，减少恢复时间。但是每次重写还是有一定的负担的，因此设定 Redis 要满足一定条件才会进行重写
    >
    > - auto-aof-rewrite-percentage：设置重写的基准值，文件达到 100% 时开始重写（文件是原来重写后文件的 2 倍时触发）
    >
    > - auto-aof-rewrite-min-size：设置重写的基准值，最小文件 64MB。达到这个值开始重写
    >
    > 例如：文件达到 70MB 开始重写，降到 50MB，下次什么时候开始重写？100MB
    >
    > ​    系统载入时或者上次重写完毕时，Redis 会记录此时 AOF 大小，设为 base_size，如果 Redis 的 AOF 当前大小 >= base_size +base_size*100% （默认）且当前大小 >=64mb （默认）的情况下，Redis 会对 AOF 进行重写
    >

  - **重写流程**

    - bgrewriteaof 触发重写，判断是否当前有 bgsave 或 bgrewriteaof 在运行，如果有，则等待该命令结束后再继续执行
    - 主进程 fork 出子进程执行重写操作，保证主进程不会阻塞
    - 子进程遍历 redis 内存中数据到临时文件，客户端的写请求同时写入 aof_buf 缓冲区和 aof_rewrite_buf 重写缓冲区保证原 AOF 文件完整以及新 AOF 文件生成期间的新的数据修改动作不会丢失
      - 子进程写完新的 AOF 文件后，向主进程发信号，父进程更新统计信息
      - 主进程把 aof_rewrite_buf 中的数据写入到新的 AOF 文件
    - 使用新的 AOF 文件覆盖旧的 AOF 文件，完成 AOF 重写
    
    <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,size_16,color_FFFFFF,t_7120-1735135b3d1b339c7cee7f909293a187-e752b6" alt="img" style="zoom:80%;" />

#### 2.4 优势与劣势

- **优势**

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/046d490cb9a34a999d1c126150ad6a2f-deacc465740f0161a96029dc9def151a-8c032f.png" alt="img" style="zoom:80%;" />

  > - 备份机制更稳健，丢失数据概率更低
  > - 可读的日志文本，通过操作AOF

- **劣势**

  > - 比起 RDB 占用更多的磁盘空间
  > - 恢复备份速度要慢
  > - 每次读写都同步的话，有一定的性能压力
  > - 存在个别 Bug，造成恢复不能

#### 2.5 总结

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/16,color_FFFFFF,t_70-68f5998be645b238a49e7587f2ff62b1-188829" alt="img" style="zoom: 80%;" />

#### 2.6 AOF和RDB的选择

官方推荐两个都启用

如果对数据不敏感，可以选单独用 RDB

不建议单独用 AOF，因为可能会出现 Bug

如果只是做纯内存缓存，可以都不用

- RDB 持久化方式能够在指定的时间间隔能对你的数据进行快照存储

- AOF 持久化方式记录每次对服务器写的操作，当服务器重启的时候会重新执行这些命令来恢复原始的数据，AOF 命令以 redis 协议追加保存每次写的操作到文件末尾.

- Redis 还能对 AOF 文件进行后台重写，使得 AOF 文件的体积不至于过大

- 只做缓存：如果你只希望你的数据在服务器运行的时候存在，你也可以不使用任何持久化方式.

- 同时开启两种持久化方式

  - 在这种情况下，当 redis 重启的时候会优先载入 AOF 文件来恢复原始的数据，因为在通常情况下 AOF 文件保存的数据集要比 RDB 文件保存的数据集要完整.

  - RDB 的数据不实时，同时使用两者时服务器重启也只会找 AOF 文件，那要不要只使用AOF呢？
    - 建议不要，因为 RDB 更适合用于备份数据库（AOF 在不断变化不好备份），快速重启，而且不会有 AOF 可能潜在的 bug，留着作为一个万一的手段

- 性能建议

  > - 因为 RDB 文件只用作后备用途，建议只在 Slave 上持久化 RDB 文件，而且只要 15 分钟备份一次就够了，只保留 save 900 1 这条规则
  >
  > - 如果使用 AOF，好处是在最恶劣情况下也只会丢失不超过两秒数据，启动脚本较简单只 load 自己的 AOF 文件就可以了
  >
  > - 代价,一是带来了持续的 IO，二是 AOF rewrite 的最后将 rewrite 过程中产生的新数据写到新文件造成的阻塞几乎是不可避免的
  >
  > - 只要硬盘许可，应该尽量减少 AOF rewrite 的频率，AOF 重写的基础大小默认值 64M 太小了，可以设到 5G 以上
  >
  > - 默认超过原大小 100% 大小时重写可以改到适当的数值

---

## 十二、Redis主从复制

### 1.概述

> 主机数据更新后根据配置和策略， 自动同步到备机的 master/slaver 机制，**Master 以写为主，Slave 以读为主**

**作用**

- 读写分离，性能扩展
- 容灾快速恢复

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/color_FFFFFF,t_70-672b4e0a0953978ebcafd576b567dda6-861724" alt="在这里插入图片描述" style="zoom:80%;" />

### 2.搭建主从复制

- 在根目录下创建文件夹 myredis，把 redis 的配置文件复制过来，要把 aof 持久化关掉

  ```bash
  mkdir myredis
  cp /etc/redis.conf /myredis/redis.conf
  ```

- 配置一主两从：创建三个文件 redis6379.conf、redis6381.conf、redis6380.conf 内容如下

  ```bash
  ######################redis6379.conf#######################
  include /myredis/redis.conf
  pidfile /var/run/redis_6379.pid
  port 6379
  dbfilename dump6379.rdb
  
  ######################redis6380.conf#######################
  include /myredis/redis.conf
  pidfile /var/run/redis_6380.pid
  port 6380
  dbfilename dump6380.rdb
  
  ######################redis6381.conf#######################
  include /myredis/redis.conf
  pidfile /var/run/redis_6381.pid
  port 6381
  dbfilename dump6381.rdb
  
  ######################额外配置##############################
  logfile "6380.log"
  #slaveof表示作为从库的配置
  slaveof 127.0.0.1 6379
  #从库只能读操作（默认）
  slave-read-only yes
  # 设置主机密码
  masterauth 主机密码
  ```

  > linux 系统中`/var/run/`目录下的 *.pid 文件是一个文本文件，其内容只有一行，即某个进程的 PID。.pid 文件的作用是防止进程启动多个副本，只有获得特定 pid 文件（固定路径和文件名）的写入权限（F_WRLCK）的进程才能正常启动并将自身的进程 PID 写入该文件，其它同一程序的多余进程则自动退出。

- 同时启动三个端口的 redis

  ```bash
  redis-server redis6379.conf
  redis-server redis6380.conf
  redis-server redis6381.conf
  ```

- 查看服务是否启动

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/1410863119fc4e30adb2d5a09baa2883-4d2c50b87fe38d9f505c6e39a384c28f-aafe36.png" alt="img" style="zoom:80%;" />

- 查看三台主机运行情况

  ```bash
  redis-cli -p port # 根据端口号连接
  info replication  # 打印主从复制的相关信息
  ```

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0FxXzQ1NDA4Mzkw,size_16,color_FFFFFF,t_70-10065086431a215738170929e384cd93-a8f270" alt="img" style="zoom:80%;" />

- 配从(库)不配主(库)

  ```bash
  slaveof <ip><port>  # 在从机上配置 主机的ip 和 端口
  # 在 6380 和 6381 上执行: slaveof 127.0.0.1 6379 -> 6379 作为 6380 和 6381 的主机
  masterauth # 配置主机密码
  ```

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/_70-8cf790e20895bbd1ebf4092daa441b35-7c4036" alt="img" style="zoom:80%;" />

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/1234-66011c61d29e43e2485132ec2c715fe6-53304c" alt="img" style="zoom:80%;" />

- 在主机上写，在从机上可以读取数据，在从机上写数据报错

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/12345-a628ea2da9a545db50060d11938fc3f2-0533b5" alt="img" style="zoom:80%;" />

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/0e78c208482b4c0db62af54bc7ca9b60-6cd1fadfadba239fd2449f6b78afc595-cc719b.png" alt="img" style="zoom:80%;" />

  

- 主机挂掉，重启就行，一切如初，从机重启需重设：slaveof 127.0.0.1 6379

### 3 常用3招

#### 3.1 一主二仆

- 切入点问题？slave1、slave2 是从头开始复制还是从切入点开始复制？比如从 k4 进来，那之前的 k1,k2,k3 是否也可以复制？

  > 从机会全量复制主机的内容，k1,k2,k3,k4 都会复制

- 从机是否可以写？set可否？

  > 从机只可读，不可写

- 主机 shutdown 后情况如何？从机是上位还是原地待命？

  > 主机 shutdown 后，从机原地待命，等待主机重新启动，一切回复正常

- 主机又回来了后，主机新增记录，从机还能否顺利复制？

  > 可以复制，因为主机重启后和之前一样，主机写内容，会同步到从机中

- 其中一台从机 down 后情况如何？依照原有它能跟上大部队吗？

  > 从机 down 后，会脱离大部队，如果重新启动，还想同步主机内容的话，需要重新执行命令 slaveof

**复制原理**

- Slave 启动成功连接到 Master 后会发送一个 sync 命令

- Master 接到命令启动后台的存盘进程，同时收集所有接收到的用于修改数据集命令，在后台进程执行完毕之后，master 将传送整个数据文件（RDB 文件）到 Slave，以完成一次完全同步

- 全量复制：Slave 服务在接收到数据库文件数据后，将其存盘并加载到内存中

- 增量复制：Master 继续将新的所有收集到的修改命令依次传给 Slave，完成同步

- 但是只要是重新连接 Master，一次完全同步（全量复制）将被自动执行

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/e1858fc3316944678d28a3e140715ffb-1db810251341cc7299d2a119fd098cd8-1edcab.png" alt="img" style="zoom:80%;" />

#### 3.2 薪火相传

> - 上一个 slave 可以是下一个 slave 的 Master，slave 同样可以接收其他 slaves 的连接和同步请求，那么该 slave 作为了链条中下一个的 master，可以有效减轻 master 的写压力，去中心化降低风险
>
> - 用 slaveof
>   - 中途变更转向：会清除之前的数据，重新建立拷贝最新的
>   - 风险是一旦某个 slave 宕机，后面的 slave 都没法备份
>   - 主机挂了，从机还是从机，无法写数据了

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/352e37d65a0e443985a4aef24d724048-e43500397464d2cf3dc0bcef41ba3923-57eced.png" alt="img" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/1-fe6b655c713540aeb3cd37b904c90342-b3c3c0" alt="img" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/2-d1c40f8065528cab41573f3c86247688-bd0da0" alt="img" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/3-f40dfc9f5587c74df385b9508b592b96-bfb702" alt="img" style="zoom:80%;" />

#### 3.3 反客为主

> 当一个 master 宕机后，后面的 slave 可以立刻升为 master，其后面的 slave 不用做任何修改
>
> 用 `slaveof no one` 将从机变为主机

- 6379 down 了

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/4-a78d9db0c9fb30fbbcb6c6a152589324-c65c71" alt="img" style="zoom:80%;" />

- 让 6380 反客为主

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/5-b0f887bc37007372d18aa98361062c6a-a4955b" alt="img" style="zoom:80%;" />

### 4 哨兵模式

> **反客为主的自动版**，能够后台监控主机是否故障，如果故障了根据投票数自动将从库转换为主库

**实例**

- 先搭建一主二从的环境
- 自定义的 /myredis 目录下新建 **sentinel.conf** 文件，名字绝不能错
- 配置哨兵，填写内容

```bash
sentinel monitor mymaster 127.0.0.1 6379 1
#其中 mymaster 为监控的主机对象起的服务器名称， 1 为至少有多少个同意主机宕机的哨兵数量
```

- 启动哨兵，执行 `redis-sentinel /myredis/sentinel.conf`

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/6-0abccb9bc00c74a79877057e50f87691-195b9f" alt="img" style="zoom:80%;" />

- 当主机挂掉，从机选举中产生新的主机

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/7-4e81fcce68d07bb65b39fa4d300164df-3dc468" alt="img" style="zoom:80%;" />

- 重新启动原主机，**原主机重启后会变为从机**

**复制延时**

由于所有的写操作都是先在 Master 上操作，然后同步更新到 Slave 上，所以从 Master 同步到 Slave 机器有一定的延迟，当系统很繁忙的时候，延迟问题会更加严重，Slave 机器数量的增加也会使这个问题更加严重。

**故障恢复**

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/8-2c16654cf1f1d79f31da411d4349c12c-9b839c" alt="img" style="zoom:80%;" />

> 优先级在 redis.conf 中设置，默认：replica-priority 100，值越小优先级越高
>
> 偏移量是指获得原主机数据最全的
>
> 每个 redis 实例启动后都会随机生成一个 40 位的 runid

**主从复制**

> Jedis 对象用这个方法取得

```java
private static JedisSentinelPool jedisSentinelPool=null;

public static  Jedis getJedisFromSentinel(){
    if(jedisSentinelPool==null){
        Set<String> sentinelSet=new HashSet<>();
        sentinelSet.add("192.168.11.103:26379");

        JedisPoolConfig jedisPoolConfig =new JedisPoolConfig();
        jedisPoolConfig.setMaxTotal(10); //最大可用连接数
        jedisPoolConfig.setMaxIdle(5); //最大闲置连接数
        jedisPoolConfig.setMinIdle(5); //最小闲置连接数
        jedisPoolConfig.setBlockWhenExhausted(true); //连接耗尽是否等待
        jedisPoolConfig.setMaxWaitMillis(2000); //等待时间
        jedisPoolConfig.setTestOnBorrow(true); //取连接的时候进行一下测试 ping pong

        jedisSentinelPool=new JedisSentinelPool("mymaster",sentinelSet,jedisPoolConfig);
        return jedisSentinelPool.getResource();
    }else{
        return jedisSentinelPool.getResource();
    }
}
```

### 5 SpringBoot配置

```properties
# 哨兵监听的 redis server 名称
spring.redis.sentinel.master=mymaster
# 哨兵的端口
spring.redis.sentinel.nodes=192.168.11.101:26379
spring.redis.password=123456
```

---

## 十三、Redis集群

### 1.集群的简介

redis 的哨兵模式基本已经可以实现高可用，读写分离 ，但是在这种模式下每台redis服务器都存储相同的数据，很浪费内存，所以在 redis3.0 上加入了 cluster 模式，实现的 redis 的分布式存储，也就是说每台 redis 节点上存储不同的内容。
Redis-Cluster 采用无中心结构，它的特点如下：

- 所有的 redis 节点彼此互联（P2P），内部使用二进制协议优化传输速度和带宽
- 节点的 fail 是通过集群中超过半数的节点检测失效时才生效
- 客户端与 redis 节点直连，不需要中间代理层，客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可
- Redis 集群实现了对 Redis 的水平扩容，即启动 N 个 redis 节点，将整个数据库分布存储在这 N 个节点中，每个节点存储总数据的 1/N
- Redis 集群通过分区（partition）来提供一定程度的可用性（availability）： 即使集群中有一部分节点失效或者无法进行通讯， 集群也可以继续处理命令请求

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/9-94bb17e057e1bb9b924e9a52852cc760-90b6ae" alt="img" style="zoom: 50%;" />

### 2.集群的搭建

> 搭建结果：制作 6 个实例，6379,6380,6381
>
>  6389,6390,6391 上下对应主从

- 删除文件夹中的全部持久化文件 rdb 或者 aof

- 新建六个配置文件，内容如下：(除了端口号不一样，其他都一样)

  ```bash
  include /myredis/redis.conf
  pidfile "/var/run/redis_6391.pid"
  port 6391
  dbfilename "dump6391.rdb"
  # 打开集群模式
  cluster-enabled yes
  # 设定节点配置文件名
  cluster-config-file nodes-6391.conf
  # 设定节点失联时间，超过该时间（毫秒），集群自动进行主从切换
  cluster-node-timeout 15000
  ```

> `:%s/6379/6380` 是 vim 的替换命令

- 启动6个服务

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/10-839ab6bde63464f3f25e7adf10f63a46-4e94da" alt="img" style="zoom:80%;" />

  要确保 nodes-xxxx.conf 生成。

- 将六个节点合成一个集群

  > 组合之前，请确保所有 redis 实例启动后，nodes-xxxx.conf 文件都生成正常

  先到 redis 的 src 目录中（需要 ruby 环境）

  ```bash
  cd  /opt/redis-6.2.1/src
  ```

  运行集成集群命令

  ```bash
  redis-cli --cluster create --cluster-replicas 1 192.168.242.110:6379 192.168.242.110:6380 192.168.242.110:6381 192.168.242.110:6389 192.168.242.110:6390 192.168.242.110:6391
  ```

  说明：ip 一定要真实 ip，不能是 localhost 或者127.0.0.1

   --replicas 1 配置集群，一台主机，一台从机，正好三组

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/11-0fd3114dd0dd55fd2bcd74d6bc7ce79e-d780de" alt="image-20220122173502725" style="zoom:67%;" />

- **查看是否集成成功**

  ```bash
  # 连接Redis
  redis-cli -c -p 6379
  # 查看集群信息
  cluster nodes
  ```

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/12-17e7a6d7cd1f0db638de1d6547fcc4a4-f2bf21" alt="img" style="zoom:80%;" />

### 3 集群操作和故障恢复

#### 3.1 集群操作

- 查看集群信息

  ```bash
  cluster nodes
  ```

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/13-17e7a6d7cd1f0db638de1d6547fcc4a4-473321" alt="img" style="zoom:80%;" />

- redis cluster 如何分配这六个节点

  > 一个集群至少要有三个主节点
  >
  > 选项 `--cluster-replicas 1` 表示我们希望为集群中的每个主节点创建一个从节点
  >
  > 分配原则尽量保证每个主数据库运行在不同的 IP 地址，每个从库和主库不在一个 IP 地址上

- 什么是slots？

  在运行集成集群命令后，会出现 ""[OK] All 16384 slots covered "。

  说明：一个 Redis 集群包含 16384 （0~16383） 个插槽（hash slot），数据库中的每个键都属于这 16384 个插槽的其中一个，集群使用公式 CRC16(key) %16384 来计算键 key 属于哪个槽。其中 CRC16(key) 语句用于计算键 key 的 CRC16 校验和 

  集群中的每个节点负责处理一部分插槽。 举个例子， 如果一个集群可以有主节点，其中：

  节点 A 负责处理 0 号至 5460 号插槽

  节点 B 负责处理 5461 号至 10922 号插槽

  节点 C 负责处理 10923 号至 16383 号插槽

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/334f25bc51ac497392a47a861695f3f6-7dcb223e18ff25ac7f30b950709e5f58-f1499b.png" alt="img" style="zoom:80%;" />

- 为什么是 16384？

  集群点越多，心跳包的消息携带的数据就越多，因此 redis 作者建议集群节点数量不要超过 1000 个，对于 1000 以内的集群数量，16384 足以。

- 在集群中录入值

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/9a10c18b4d714794b3c6cfd0be45214f-0b14ffcf8cdffb50d20fb604724c1fd0-b2edaf.png" alt="img" style="zoom:80%;" />

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/94006ddd9ef540c68d5015167bdd0b43-318c3f20aa55a5e8ad4eefd105a7f8fb-d5cf9c.png" alt="img" style="zoom:80%;" />

  > **注意：**在用 mset 同时设置多个值的时候，需要把这些 key 放到同一个组中，不然会报错。可以通过 {} 来定义组的概念，从而使 key 中 {} 内相同内容的键值对放到一个 slot 中去
  >
  > `mset fieldName1{group} fieldValue1 fieldName{group} fieldValue2 ...`

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/86e56a7080914797a34a178f211f8db7-58bfb85d8cb70ef2f06104ab21f2a3e6-15cdee.png" alt="img" style="zoom:80%;" />

- **查询集群中的值**

  ```bash
  cluster keyslot k1 # 查询k1的插槽值
  ```

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/9ed0fcb60b9f47d68134c3d4189ffaf9-bc2d19537024512cc1ea5c992f64b804-bb1d16.png" alt="img" style="zoom:80%;" />

  ```bash
  cluster countkeysinslot 12706 
  # 查看指定插槽中的key数量，注意只能在插槽值所在的主机上能成功，例如：12706 插槽在 6381 端口的主机上则只能在 6381 端口的主机上查到，在其他端口则查询失败
  ```
  
  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/cfb702f6cd1b46bf821d3df0f41fb699-416e383638f3442c3a643f8cc9266ce8-803de7.png" alt="img" style="zoom:80%;" />
  
  ```bash
  cluster getkeysinslot 5474 2 # 返回指定插槽的指定数量的key
  ```
  
  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/bdea7f4d54684a70a2dc6ea1f6a27883-aa718a72898b8365791244eb6c100bc7-eafdda.png" alt="img" style="zoom:80%;" />

#### 3.2 故障恢复

- 如果主节点下线？从节点能否自动升为主节点？注意：**15秒超时**，15秒内恢复连接则还是主机，否则原从机晋升为主机

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/f0edf11f6fe64e868302ce7b818762b2-87348e11e7c825a1f86fa6c8dbaca97b-6d9dbe.png" alt="img" style="zoom:80%;" />

- 主节点恢复后，主从关系会如何？
  - 主节点回来变成从机

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/7c956da3f1ee49688a9e2c3a944c7e9b-4b6c1813c2ee512fbabbfce6ff3bce00-3b5ee3.png" alt="img" style="zoom:80%;" />

- 如果所有某一段插槽的主从节点都宕掉，redis服务是否还能继续?

> - 如果某一段插槽的主从都挂掉，而 cluster-require-full-coverage 为 yes ，那么 ，整个集群都挂掉
>
> - 如果某一段插槽的主从都挂掉，而 cluster-require-full-coverage 为 no ，那么，该插槽数据全都不能使用，也无法存储，其他插槽可以使用
>
> - redis.conf 中的参数 cluster-require-full-coverage

- 从机在集群中充当“冷备”，不能缓解读压力

#### 3.3 集群的Jedis开发

> 即使连接的不是主机，集群会自动切换主机存储。主机写，从机读。
>
> 无中心化主从集群。无论从哪台主机写的数据，其他主机上都能读到数据。

```java
public class JedisClusterTest {
    public static void main(String[] args) {
        HostAndPort hostAndPort = new HostAndPort("192.168.242.110", 6381);
        JedisCluster jedisCluster = new JedisCluster(hostAndPort);
        jedisCluster.set("k5","v5");
        String k5 = jedisCluster.get("k5");
        System.out.println(k5);
    }
}
```

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/63518e61becb471694009257e8b854aa-8e1045f4eea86d4708e88e3547726ce3-fe4b4e.png" alt="img" style="zoom:80%;" />

#### 3.4 SpringBoot配置

> ```properties
> spring.redis.cluster.nodes=192.168.159.129:7001,192.168.159.129:7002,192.168.159.129:7003,192.168.159.129:7004,192.168.159.129:7005,192.168.159.129:7006
> ```

### 4 Redis的好处与不足

**好处**

- 实现扩容
- 分摊压力
- 无中心配置相对简单

**不足**

- 多键操作是不被支持的
- 多键的 Redis 事务是不被支持的，lua 脚本不被支持
  - 因为 Redis 要求单个  Lua 脚本操作的 key 必须在同一个节点上，可以使用 {} 分组功能保证拥有同样的 {}  内部字符串的 key 拥有相同的插槽

- 由于集群方案出现较晚，很多公司已经采用了其他的集群方案，而代理或者客户端分片的方案想要迁移至 redis cluster，需要整体迁移而不是逐步过渡，复杂度较大

---

## 十四、Redis应用问题的解决

### 1.缓存穿透

#### 1.1 问题描述

> ​    key 对应的数据在数据源并不存在，每次针对此 key 的请求从缓存获取不到，请求都会压到数据源，从而可能压垮数据源。比如用一个不存在的用户 id 获取用户信息，不论缓存还是数据库都没有，若黑客利用此漏洞进行攻击可能压垮数据库。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/66-126b76aeeee37e32c6ddfa83ad47cda4-acf56f" alt="img" style="zoom:80%;" />

#### 1.2 解决方案

> 一个一定不存在缓存及查询不到的数据，由于缓存是不命中时被动写的，并且出于容错考虑，如果从存储层查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到存储层去查询，失去了缓存的意义。

**解决方案：**

- 对空值缓存：如果一个查询返回的数据为空（不管是不是由于数据不存在），我们仍然把这个空结果（null）进行缓存（比如设置 value 为 -1，代表没有数据），设置空结果的过期时间会很短，最长不超过五分钟

- 设置可访问的名单（白名单）：使用 bitmaps 类型定义一个可以访问的名单，名单 id 作为 bitmaps 的偏移量，每次访问和 bitmap 里面的 id 进行比较，如果访问 id 不在 bitmaps 里面，进行拦截，不允许访问，可使用`stringRedisTemplate.opsForValue().setBit(key, hash, value)`命令

- 采用布隆过滤器：布隆过滤器（Bloom Filter）是1970年由布隆提出的。它实际上是一个很长的二进制向量（位图）和一系列随机映射函数（哈希函数）。布隆过滤器可以用于检索一个元素是否在一个集合中。它的优点是空间效率和查询时间都远远超过一般的算法，缺点是有一定的误识别率和删除困难。将所有可能存在的数据哈希到一个足够大的 bitmaps 中，一个一定不存在的数据会被 这个 bitmaps 拦截掉，从而避免了对底层存储系统的查询压力

- 进行实时监控：当发现的命中率开始急速降低，需要排查访问对象和访问的数据，和运维人员配合，可以设置黑名单限制服务

### 2.缓存击穿

#### 2.1 问题描述

> key 对应的数据存在，但在 redis 中过期，此时若有大量并发请求过来，这些请求发现缓存过期一般都会从后端 DB 加载数据并回设到缓存，这个时候大并发的请求可能会瞬间把后端DB压垮。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/77-1bccd278163e13c26f3a4bcdaf957891-6b1a31" alt="img" style="zoom:80%;" />

#### 2.2 解决方案

> key 可能会在某些时间点被超高并发地访问，是一种非常"热点"的数据。这个时候，需要考虑一个问题：缓存被“击穿”的问题。

**解决方案**

- 预先设置热门数据：在 redis 高峰访问之前，把一些热门数据提前存入到 redis 里面，加大这些热门数据 key 的时长

- 实时调整：现场监控哪些数据热门，实时调整 key 的过期时长

- 使用锁：
  - 就是在缓存失效的时候（判断拿出来的值为空），不是立即去 load db
  - 先使用缓存工具的某些带成功操作返回值的操作（比如 Redis 的 SETNX）去 set 一个 mutex key
  - 当操作返回成功时，再进行 load db 的操作，并回设缓存，最后删除 mutex key
  - 当操作返回失败，证明有线程在 load db，当前线程睡眠一段时间再重试整个 get 缓存的方法

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/88-849ac892c8530ccc3fb4db3e8f72587f-1c0b44" alt="img" style="zoom:80%;" />

### 3.缓存雪崩

#### 3.1 问题描述

> ​    key 对应的数据存在，但在 redis 中过期，此时若有大量并发请求过来，这些请求发现缓存过期一般都会从后端 DB 加载数据并回设到缓存，这个时候大并发的请求可能会瞬间把后端 DB 压垮。缓存雪崩与缓存击穿的区别在于这里针对很多 key 缓存，前者则是某一个 key 正常访问

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/99-01593c763ea28451367ac326f9e265f4-79f806" alt="img" style="zoom:80%;" />

#### 3.2 解决方案

- 构建多级缓存架构：nginx缓存 + redis缓存 +其他缓存（ehcache 等）
- 使用锁或队列：用加锁或者队列的方式保证来保证不会有大量的线程对数据库一次性进行读写，从而避免失效时大量的并发请求落到底层存储系统上。不适用高并发情况
- 设置过期标志更新缓存：记录缓存数据是否过期（设置提前量），如果过期会触发通知另外的线程在后台去更新实际 key 的缓存
- 将缓存失效时间分散开：比如我们可以在原有的失效时间基础上增加一个随机值，比如1-5

### 4 分布式锁

> ​    随着业务发展的需要，原单体单机部署的系统被演化成分布式集群系统后，由于分布式系统多线程、多进程并且分布在不同机器上，这将使原单机部署情况下的并发控制锁策略失效，单纯的Java API并不能提供分布式锁的能力。为了解决这个问题就需要一种跨JVM的互斥机制来控制共享资源的访问，这就是分布式锁要解决的问题。
>
> 分布式锁主流的实现方案：
>
> - 基于数据库实现分布式锁
> - 基于缓存（Redis等）
> - 基于 Zookeeper
>
> 每一种分布式锁解决方案都有各自的优缺点：
>
> - 性能：redis 最高
> - 可靠性：zookeeper 最高

**优化设置锁和过期时间**

1. 设置锁的命令

```bash
SETNX KEY VALUE  # 设置锁
del key   # 删除锁
```

2. 给锁设置过期时间

```bash
expire users 30
```

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/111-264ac3f661bb6ab89a57b0ffd1d74179-772d02" alt="img" style="zoom:80%;" />

> 这样设置的问题：如果设置时间和上锁分开进行的话，可能存在上完锁，服务器 down 了，就没有设置过期时间。

==优化：上锁和设置过期时间同时==

```bash
set key value nx ex time
```

**java代码实现**：将 num 依次加 1

```java
@GetMapping("testLock")
public void testLock(){
    //1 获取锁，setnx ,顺便设置过期时间
    Boolean lock = redisTemplate.opsForValue().setIfAbsent("lock", "111", 3, TimeUnit.SECONDS);
    //2 获取锁成功、查询 num 的值
    if(lock){
        Object value = redisTemplate.opsForValue().get("num");
        //2.1 判断 num 为空 return
        if(StringUtils.isEmpty(value)){
            return;
        }
        //2.2 有值就转成 int
        int num = Integer.parseInt(value+"");
        //2.3 把 redis 的 num 加 1
        redisTemplate.opsForValue().set("num", ++num);
        //2.4 释放锁，del
        redisTemplate.delete("lock");

    }else{
        //3 获取锁失败、每隔 0.1 秒再获取
        try {
            Thread.sleep(100);
            testLock();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

**优化之UUID防止误删**

不过代码除了修改的设置过期时间问题，还存在问题，入下图所示：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/12332-0245e4507f9b33a3d627bd3810e576bb-011a5d" alt="img" style="zoom:80%;" />

解决方法：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/123456-733d3189fc225ec9ea0fab2ccde884ac-56502f" alt="img" style="zoom:80%;" />

代码实现：

```java
@GetMapping("testLock")
public void testLock(){
	String uuid = UUID.randomUUID().toString();
    //1获取锁，setne ,顺便设置过期时间
    Boolean lock = redisTemplate.opsForValue().setIfAbsent("lock", uuid,3,TimeUnit.SECONDS);
    //2获取锁成功、查询num的值
    if(lock){
       ...
        String lockUuid = (String)redisTemplate.opsForValue().get("lock");
        if(uuid.equals(lockUuid)){
             //2.4释放锁，del
        	redisTemplate.delete("lock");
        }
    }else{
       ...
    }
}
```

##### ==优化之LUA脚本保证删除的原子性==

[Lua](./Lua.md)

原因：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/a-dd3ba5ffe823fdeea6293d38e1f69aa1-ffd425" alt="img" style="zoom:67%;" />

解决方案：使用lua脚本保证删除的原子性

```java
@GetMapping("testLockLua")
public void testLockLua() {
    //1 声明一个uuid ,将做为一个value 放入我们的key所对应的值中
    String uuid = UUID.randomUUID().toString();
    //2 定义一个锁：lua 脚本可以使用同一把锁，来实现删除！
    String skuId = "25"; // 访问skuId 为25号的商品 100008348542
    String locKey = "lock:" + skuId; // 锁住的是每个商品的数据

    // 3 获取锁
    Boolean lock = redisTemplate.opsForValue().setIfAbsent(locKey, uuid, 3, TimeUnit.SECONDS);

    // 第一种： lock 与过期时间中间不写任何的代码。
    // redisTemplate.expire("lock",10, TimeUnit.SECONDS);//设置过期时间
    // 如果true
    if (lock) {
        /// 执行的业务逻辑开始
        // 获取缓存中的num 数据
        Object value = redisTemplate.opsForValue().get("num");
        if (StringUtils.isEmpty(value)) {
            return;
        }
        int num = Integer.parseInt(value + "");
        redisTemplate.opsForValue().set("num", String.valueOf(++num));
        
        /// 使用lua脚本来锁
        // 定义lua 脚本
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        // 使用redis执行lua执行
        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptText(script);
        // 设置一下返回值类型 为Long
        // 因为删除判断的时候，返回的0,给其封装为数据类型。如果不封装那么默认返回 String 类型，
        // 那么返回字符串与 0 会有发生错误。
        redisScript.setResultType(Long.class);
        // 第一个要是script 脚本 ，第二个需要判断的key，第三个就是key所对应的值。
        redisTemplate.execute(redisScript, Arrays.asList(locKey), uuid);
    } else {
        // 其他线程等待
        try {
            // 睡眠
            Thread.sleep(1000);
            // 睡醒了之后，调用方法。
            testLockLua();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

### **总结**

为了确保分布式锁可用，我们至少要确保锁的实现同时满足以下四个条件：

- 互斥性。在任意时刻，只有一个客户端能持有锁
- 不会发生死锁。即使有一个客户端在持有锁的期间崩溃而没有主动解锁，也能保证后续其他客户端能加锁
- 解铃还须系铃人。加锁和解锁必须是同一个客户端，客户端自己不能把别人加的锁给解了
- 加锁和解锁必须具有原子性

---

## 十五、Redis6.0的新功能

### 1.ACL

#### 1.1 简介

> Redis ACL 是 Access Control List（访问控制列表）的缩写，该功能允许根据可以执行的命令和可以访问的键来限制某些连接。
>
> 在Redis 5 版本之前，Redis 安全规则只有密码控制，还有通过 rename 来调整高危命令比如 flushdb ，KEYS* ，shutdown 等。Redis 6 则提供 ACL 的功能对用户进行更细粒度的权限控制：
>
> - 接入权限：用户名和密码
> - 可以执行的命令
> - 可以操作的 KEY

#### 1.2 命令

- 使用 `acl list` 命令展现用户权限列表

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/f6ef6cd7875b46fb8419e9ba054d24be-e892977e4904c6d0cd266d6ef6cf891b-701b35.png" alt="img" style="zoom:80%;" />

- 使用 `acl cat` 命令

  - 查看添加权限指令类别

    <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/d90cfb671048494fb06a42deb8f7e6d3-203f1240afc49bf2290e8c934ee161a4-5701f9.png" alt="img" style="zoom:80%;" />

  - 加参数类型名可以查看类型下具体命令

    <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/b8104e4448a74ef5837e6825b33f6902-7fc8dc8a8cd095ef554608e70d0b0a4c-6e05dd.png" alt="img" style="zoom:80%;" />

- 使用 `acl whoami` 命令查看当前用户

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/db1aba8f89704486bb1e9efd01cf6f31-fae00f7e55e55a5a9c408dc206846151-db8bc0.png" alt="img"  />

- 使用 aclsetuser 命令创建和编辑用户 ACL

  - ACL规则

    > 下面是有效 ACL 规则的列表。某些规则只是用于激活或删除标志，或对用户 ACL 执行给定更改的单个单词。其他规则是字符前缀，它们与命令或类别名称、键模式等连接在一起
    >
    > |         类型         |        参数        |                             说明                             |
    > | :------------------: | :----------------: | :----------------------------------------------------------: |
    > |    启动和禁用用户    |       **on**       |                        激活某用户账号                        |
    > |    启动和禁用用户    |      **off**       | 禁用某用户账号。注意，已验证的连接仍然可以工作。如果默认用户被标记为off，则新连接将在未进行身份验证的情况下启动，并要求用户使用AUTH选项发送AUTH或HELLO，以便以某种方式进行身份验证。 |
    > |    权限的添加删除    |    +`<command>`    |             将指令添加到用户可以调用的指令列表中             |
    > |    权限的添加删除    |    -`<command>`    |                 从用户可执行指令列表移除指令                 |
    > |    权限的添加删除    | **+@**`<categroy>` | 添加该类别中用户要调用的所有指令，有效类别为@admin、@set、@sortedset…等，通过调用ACL CAT命令查看完整列表。特殊类别@all表示所有命令，包括当前存在于服务器中的命令，以及将来将通过模块加载的命令 |
    > |    权限的添加删除    |   -@`<actegory>`   |                  从用户可调用指令中移除类别                  |
    > |    权限的添加删除    |  **allcommands**   |                         +@all的别名                          |
    > |    权限的添加删除    |   **nocommand**    |                         -@all的别名                          |
    > | 可操作键的添加或删除 | **~** `<pattarn>`  |     添加可作为用户可操作的键的模式。例如 ~* 允许所有的键     |

  - 通过命令创建新用户默认权限

    ```bash
    acl setuser user1
    ```

    <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/72dd8294941e452fbc573d3633d82a1c-194f51f60c7eca3e5a98719ce4cb40a6-94154a.png" alt="img" style="zoom:80%;" />

    > 在上面的示例中，我根本没有指定任何规则。如果用户不存在，这将使用 just created 的默认属性来创建用户。如果用户已经存在，则上面的命令将不执行任何操作。

  - 设置有用户名、密码、ACL 权限、并启用的用户

    ```bash
    acl setuser user2 on >password ~cached:* +get # 只能 get 以 cached: 开头的 key
    ```

    <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/beaea2f2692447238d58336a0fa01bfe-25d013389a155ae1e80adb186d0c776d-81358a.png" alt="img"  />

  - 切换用户，验证权限

    <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/0d0fd42952cf4553be17a1d6c7a7166f-4846b5a7947c27880eb2d5eb798f9d8a-a106a8.png" alt="img"  />

### 2 IO多线程

> IO 多线程其实指**客户端交互部分**的**网络IO**交互处理模块**多线程**，而非**执行命令多线程**，Redis6 执行命令依然是单线程

**原理架构**

​    Redis 6 加入多线程,但跟 Memcached 这种从 IO 处理到数据访问多线程的实现模式有些差异。Redis 的多线程部分只是用来处理网络数据的读写和协议解析，执行命令仍然是单线程。之所以这么设计是不想因为多线程而变得复杂，需要去控制 key、lua、事务，LPUSH/LPOP 等等的并发问题。整体的设计大体如下：
<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,s_16,color_FFFFFF,t_70-61465d8de75b35087f550923b6168cd7-1fbac7" alt="img" style="zoom:150%;" />

> 另外，多线程 IO 默认也是不开启的，需要再配置文件中配置
>
> io-threads-do-reads yes
>
> io-threads 4

### 3 工具支持 Cluster

​    之前老版 Redis 想要搭集群需要单独安装 ruby 环境，Redis 5 将 redis-trib.rb 的功能集成到 redis-cli 。另外官方 redis-benchmark 工具开始支持 cluster 模式了，通过多线程的方式对多个分片进行压测压。

![img](https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NDA4Mzkw,size_16,color_FFFFFF,-8e77f934bd4da0ffa59838205c62bf7e-d8465b)

> Redis6新功能还有：
>
> - RESP3 新的 Redis 通信协议：优化服务端与客户端之间通信
> - Client side caching 客户端缓存：基于 RESP3 协议实现的客户端缓存功能。为了进一步提升缓存的性能，将客户端经常访问的数据 cache 到客户端。减少 TCP 网络交互
> - Proxy 集群代理模式：Proxy 功能，让 Cluster 拥有像单实例一样的接入方式，降低大家使用 cluster 的门槛。不过需要注意的是代理不改变 Cluster 的功能限制，不支持的命令还是不会支持，比如跨 slot 的多Key操作
> - Modules API：Redis 6 中模块开发进展非常大，因为为了开发复杂的功能，从一开始就用上模块。可以变成一个框架，利用来构建不同系统，而不需要从头开始写然后还要许可。一开始就是一个向编写各种系统开放的平台
