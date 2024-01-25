# 第14章_MySQL事务日志

事务有4种特性：原子性、一致性、隔离性和持久性。那么事务的四种特性到底是基于什么机制实现呢？

- 事务的隔离性由`锁机制`实现
- 而事务的原子性、一致性和持久性由事务的 redo 日志和 undo 日志来保证
  - REDO LOG 称为`重做日志`，提供再写入操作，恢复提交事务修改的页操作，用来保证事务的`持久性`
  - UNDO LOG 称为`回滚日志`，回滚行记录到某个特定版本，用来保证事务的`原子性`、`一致性`

有的 DBA 或许会认为 UNDO 是 REDO 的逆过程，其实不然。REDO 和 UNDO 都可以视为一种`恢复操作`，但是：

- redo log：是存储引擎层（InnoDB）生成的日志，记录的是`物理级别`上的页修改操作，比如页号 XXX，偏移量 YYY 写入了 ZZZ 数据，主要为了保证数据的可靠性
- undo log：是存储引擎层（InnoDB）生成的日志，记录的是`逻辑操作`日志，比如对某一行数据进行了 INSERT 语句操作，那么 undo log 就记录一条与之相反的 DELETE 操作，主要用于`事务的回滚`（undo log 记录的是每个修改操作的`逆操作`）和`一致性非锁定读`（undo log 回滚行记录到某种特性的版本...MVCC，即多版本并发控制）

## 1.redo日志

InnoDB 存储引擎是以`页为单位`来管理存储空间的。在真正访问页面之前，需要把在`磁盘上`的页缓存到内存中的`Buffer poll`之后才可以访问。所有的变更都必须`先更新缓冲池`中的数据，然后缓冲池中的`脏页`（内存中被修改的数据还没有被刷新到磁盘上）会以一定的频率被刷入磁盘（`checkpoint机制`），通过缓冲池来优化 CPU 和磁盘之间的鸿沟，这样就可以保证整体的性能不会下降太快。

### 1.1 为什么需要REDO日志

一方面，缓冲池可以帮助我们消除 CPU 和磁盘之间的鸿沟，checkpoint 机制可以保证数据的最终落盘，然而由于checkpoint `并不是每次变更的时候就触发`的，而是 master 线程隔一段时间去处理的。所以最坏的情况就是事务提交后，刚写完缓冲池，数据库宕机了，那么这段数据就是丢失的，无法恢复。

另一方面，事务包含`持久性`的特性，就是说对于一个已经提交的事务，在事务提交后即使系统发生了崩溃，这个事务对数据库中所做的更改也不能丢失。

那么如何保证这个持久性呢？`一个简单的做法`：在事务提交完成之前把该事务所修改的所有页面都刷新到磁盘，但是这个简单粗暴的做法有些问题：

- **修改量与刷新磁盘工作量严重不成比例**

  有时候我们仅仅修改了某个页面中的一个字节，但是我们知道 InnoDB 中是以页为单位来进行磁盘 IO 的，也就是说我们在该事务提交时不得不将一个完整的页面从内存中刷新到磁盘，我们又知道一个页面默认是 16KB，只修改一个字节就要刷新 16KB 的数据到磁盘上显然是大题小作了。

- **随机IO刷新较慢**

  一个事务可能包含很多语句，即使一条语句也可能修改许多页面，假如该事务修改的这些页面可能并不相邻，这就意味着在将某个事务修改的 Buffer Pool 中的页面`刷新到磁盘`时，需要进行很多的`随机IO`，随机 IO 比顺序 IO 慢很多，尤其对于传统的机械硬盘来说。

`另一个解决的思路`：我们只是想让已经提交了的事务对数据库中数据所做的修改永久生效，即使后来系统崩溃，在重启后也能把这种修改恢复出来。所以我们其实没有必要在每次事务提交时就把该事务在内存中修改过的全部页面刷新到磁盘，只需要把`修改 `了哪些东西`记录一下`就好。比如，某个事务将系统表空间中`第10号`页面中偏移量为 100 处的那个字节的值 1 改成 2。我们只需要记录一下：将第 0 号表空间的 10 号页面的偏移量为 100 处的值更新为 2 。

InnoDB 引擎的事务采用了 WAL 技术（`Write-ahead Logging`），这种技术的思想就是先写日志，再写磁盘，只有日志写入成功，才有事务提交成功，这里的日志就是 redo log。当发生宕机且数据未刷新到磁盘的时候，可以通过 redo log 来恢复，保证 ACID 中的 D，这就是 redo log 的作用。

<img src="img\image-20220223215637710.png" alt="image-20220223215637710" style="zoom:67%;" />

### 1.2 REDO日志的好处、特点

#### 1.好处

- **redo日志降低了刷盘频率**
- **redo日志占用的空间非常小**

存储表空间 ID。页号。偏移量以及需要更新的值，所需的存储空间很小，刷盘快。

#### 2.特点

- **redo日志是顺序写入磁盘的**

  在执行事务的过程中，每执行一条语句，就可能产生若干条 redo 日志，这些日志是按照`产生的顺序写入磁盘的`，也就是使用顺序 IO，效率比随机 IO 快。

  > 注意：之后刷新磁盘上的数据时还是随机 IO

- **事务执行过程中，redo log不断记录**

  redo log 跟 bin log 的区别，redo log 是`存储引擎层`产生的，而 bin log 是`数据库层`产生的。假设一个事务，对表的 10 万行的记录插入，在这个过程中，一直不断的往 redo log 顺序记录，而 bin log 不会记录，直到这个事务提交，才会一次写入到 bin log 文件中。
  
- **记录的是磁盘上的偏移量，不能用于主从复制，因为不同服务器的数据在磁盘上存储位置不同**

### 1.3 redo的组成

Redo log可以简单分为以下两个部分：

- `重做日志的缓冲（redo log buffer）`，保存在内存中，是易失的

  在服务器启动时就向操作系统申请了一大片称之为 redo log buffer 的`连续内存`空间，翻译成中文就是 redo 日志缓冲区。这片内存空间被划分为若干个连续的`redo log block`。一个 redo log block 占用`512字节`大小。

  <img src="img\image-20220224232053395.png" alt="image-20220224232053395" style="zoom:67%;" />

  **参数设置：innodb_log_buffer_size：**

  redo log buffer 大小，默认 16M，最大值是4096M，最小值为1M。

  ```sql
  mysql> show variables like '%innodb_log_buffer_size%';
  +------------------------+----------+
  | Variable_name     | Value  |
  +------------------------+----------+
  | innodb_log_buffer_size | 16777216 |
  +------------------------+----------+
  ```

- `重做日志文件 (redo log file)`，保存在硬盘中，是持久的

  REDO 日志文件如图所示，其中的`ib_logfile0`和`ib_logfile1`即为 REDO 日志。

  <img src="img\image-20220224233118737.png" alt="image-20220224233118737" style="zoom:67%;" />

### 1.4 redo的整体流程

以一个更新事务为例，redo log 流转过程，如下图所示：

<img src="img\image-20220223215705064.png" alt="image-20220223215705064" style="zoom: 50%;" />

> 第1步：先将原始数据从磁盘中读入内存中来，修改数据的内存拷贝
> 第2步：生成一条重做日志并写入`redo log buffer`，记录的是数据被修改后的值
> 第3步：当事务 commit 时，将 redo log buffer 中的内容刷新到 redo log file，对 redo log file 采用追加写的方式
> 第4步：定期将内存中修改的数据刷新到磁盘中

> 体会：
>
> Write-Ahead Log（预先日志持久化）：在持久化一个数据页之前，先将内存中相应的日志页持久化。

### 1.5 redo log的刷盘策略

redo log 的写入并不是直接写入磁盘的，InnoDB 引擎会在写 redo log 的时候先写 redo log buffer，之后以`一定的频率`刷入到真正的 redo log file 中。这里的一定频率怎么看待呢？这就是我们要说的刷盘策略。（先写入对应内存的开辟的空间 redo log buffer，然后根据某个策略写入磁盘 redo log file）

<img src="img\image-20220223215719887.png" alt="image-20220223215719887" style="zoom:67%;" />

注意，redo log buffer 刷盘到 redo log file 的过程并不是真正的刷到磁盘中去，只是刷入到`文件系统缓存（page cache）`中去（这是现代操作系统为了提高文件写入效率做的一个优化），真正的写入会交给系统自己来决定（比如 page cache 足够大了）。那么对于 InnoDB 来说就存在一个问题，如果交给系统来同步，同样如果系统宕机，那么数据也丢失了（虽然整个系统宕机的概率还是比较小的）。

针对这种情况，InnoDB 给出<a id="innodb_flush_log_at_trx_commit参数">`innodb_flush_log_at_trx_commit`</a>==全局==参数，该参数控制 commit 提交事务时，如何将 redo log buffer 中的日志刷新到 redo log file 中。它支持三种策略：

- `设置为0`：提交事务的时候，不立即把 redo log buffer 里的数据刷入磁盘文件的，而是依靠 InnoDB 的主线程每秒执行一次刷新到磁盘（写入 page cache 中并通知文件系统进行刷盘操作）。此时可能你提交事务了，结果 mysql 宕机了，然后此时内存里的数据全部丢失
- `设置为1`：表示每次事务提交时都将进行同步，刷盘操作（`默认值`）
- `设置为2`：表示每次事务提交时都只把 redo log buffer 内容写入 page cache（操作系统的缓存），每隔 1 秒进行一次同步

```sql
mysql> show variables like 'innodb_flush_log_at_trx_commit';
+--------------------------------+-------+
| Variable_name                  | Value |
+--------------------------------+-------+
| innodb_flush_log_at_trx_commit | 1     |
+--------------------------------+-------+
```

另外，InnoDB 存储引擎有一个后台线程，每隔 1s，就会把`redo log buffer`中的内容写到文件系统缓存`page cache`，然后调用刷盘操作。

<img src="img\image-20220224234705362.png" alt="image-20220224234705362" style="zoom:67%;" />

也就是说。一个没有提交事务的`redo log`记录，也可能会刷盘。因为在事务执行过程 redo log 记录是会写入`redo log buffer`中，这些 redo log 记录会被`后台线程`刷盘。

<img src="img\image-20220224234908005.png" alt="image-20220224234908005" style="zoom:67%;" />

除了后台线程每秒一次的轮询操作，还有一种情况，当`redo log buffer`占用的空间即将达到`innodb_log_buffer_size`（默认 16KB）的一半的时候，后台线程会主动刷盘。

### 1.6 不同刷盘策略演示

#### 1.流程图

```sql
Innodb_flush_log_at_trx_commit = 1;
```

<img src="img\image-20220223215757969.png" alt="image-20220223215757969" style="zoom:67%;" />

> 每次事务提交时 MySQL 都会把 log buffer 的数据写入 page cache，并且 flush（刷到磁盘中去）。
>
> 如果事务执行期间`MySQL`挂了或宕机，这部分日志丢失了，但是事务并没有提交，所以日志丢了也不会有损失。可以保证 ACID 中的 D，数据绝对不会丢失，但是`效率最差`。
>
> 建议使用默认值，虽然操作系统宕机的概率理论小于数据库宕机的概率，但是一般既然使用了事务，那么数据的安全相对来说更重要些。

```sql
Innodb_flush_log_at_trx_commit = 2;
```

<img src="img\image-20220223215900223.png" alt="image-20220223215900223" style="zoom:67%;" />

> 只要事务提交成功，`redo log buffer`中的内容只会写入文件系统缓存`page cache`。但是 flush（刷到磁盘）操作并不会同时进行。
>
> 如果仅仅只是 MySQL 挂了不会有任何数据丢失，但是操作系统宕机可能会有 1s 数据的丢失，这种情况下无法满足 ACID 中的 D。但是效率最高。

```sql
Innodb_flush_log_at_trx_commit = 0;
```

<img src="img\image-20220223215912197.png" alt="image-20220223215912197" style="zoom:67%;" />

> log buffer 中的内容每秒一次地写入`page cache`中，并且执行 flush（刷到磁盘），因此最多丢失 1s 内的事务。
>
> 数值 0 的话是一种折中的做法，它的 IO 效率理论是高于 1，低于 2 的，这种策略也有丢失数据的风险，也无法保证 D。

#### 2.举例

比较 innodb_flush_log_at_trx_commit 对事务的影响：

```sql
CREATE TABLE test_load(
	a INT,
    b CHAR(80)
)ENGINE=INNODB;
```

```sql
DELIMITER //
CREATE PROCEDURE p_load(COUNT INT UNSIGNED)
BEGIN
    DECLARE s INT UNSIGNED DEFAULT 1;
    DECLARE c CHAR(80) DEFAULT REPEAT('a', 80);
    WHILE s <= COUNT DO
		INSERT INTO test_load SELECT NULL, c;
		COMMIT;
		SET s = s + 1;
    END WHILE;
END //
DELIMITER ;
```

存储过程代码中，每插入一条数据就进行一次显示的 COMMIT 操作，在默认情况下，即参数 innodb_flush_log_at_trx_commit 为 1 的情况下，InnoDB 存储引擎会将重做日志缓冲中的日志写入文件，并调用一次 fsync 操作。

执行命令`CALL p_load(30000)`，向表中插入 3 万条记录，并执行 3 万次的 fsync 操作，在默认情况下耗时：

```sql
mysql> show variables like 'innodb_flush_log_at_trx_commit';
+--------------------------------+-------+
| Variable_name                  | Value |
+--------------------------------+-------+
| innodb_flush_log_at_trx_commit | 1     |
+--------------------------------+-------+

mysql> call p_load(30000);
Query OK, 0 rows affected (34.97 sec)
```

修改参数 innodb_flush_log_at_trx_commit 为 0：

```sql
mysql> set GLOBAL innodb_flush_log_at_trx_commit = 0;
Query OK, 0 rows affected (0.00 sec)

mysql> call p_load(30000);
Query OK, 0 rows affected (13.74 sec)
```

修改参数 innodb_flush_log_at_trx_commit 为 2：

```sql
mysql> set GLOBAL innodb_flush_log_at_trx_commit = 2;
Query OK, 0 rows affected (0.00 sec)

mysql> call p_load(30000);
Query OK, 0 rows affected (24.03 sec)
```

可以发现参数 innodb_flush_log_at_trx_commit 设置为 0 时拥有最快的效率，因为此时 fsync 的次数会少很多，针对这种情况，为了提高事务的提交性能，应该将 3 万行记录插入表后进行一次的 COMMIT 操作，而不是每插入一条记录后进行一次 COMMIT 操作。这样可以使事务方法在 rollback 时回滚到事务最开始的确定状态。

> 虽然用户可以通过设置参数 innodb_flush_log_at_trx_commit 为 0 或 2 来提高事务提交的性能，但需清楚，这种设置方法`丧失了事务的 ACID 特性`。

### 1.7 写入redo log buffer过程

#### 1.补充概念：Mini-Transaction

MySQL 把对底层页面中的一次原子访问的过程称之为一个`Mini-Transaction`，简称`mtr`，比如向某个索引对应的 B+ 数中插入一条记录的过程就是一个`Mini_transaction`。一个所谓的`mtr`可以包含`一组redo日志`（插入数据时需要移动其他数据），在进行崩溃恢复时这一组`redo`日志作为一个不可分割的整体。

一个事务可以包含若干条语句，每一条语句其实是由若干个`mtr`组成，每一个`mtr`又可以包含若干条 redo 日志，画个图表示它们的关系就是这样：

<img src="img\image-20220223215931652.png" alt="image-20220223215931652" style="zoom: 50%;" />

#### 2.redo日志写入log buffer

向`log buffer`中写入 redo 日志的过程是顺序的，也就是先往前边的 block 中写，当该 block 的空闲空间用完之后再往下一个 block 中写。当我们想往`log buffer`中写入 redo 日志时，第一个遇到的问题就是应该在哪个`block`的哪个偏移量处，所以`InnoDB`的设计者特意提供了一个称之为`buf_free`的全局变量，该变量指明后续写入的 redo 日志应该写入到`log buffer`中的哪个位置，如图所示：

<img src="img\image-20220223215951592.png" alt="image-20220223215951592" style="zoom:67%;" />

一个 mtr 执行过程中可能产生若干条 redo 日志，`这些redo日志是一个不可分割的组`，所以其实并不是每生成一条 redo 日志，就将其插入到 log buffer 中，而是每个 mtr 运行过程中产生的日志先暂时存到一个地方，当该 mtr 结束的时候，将过程中产生的一组 redo 日志在全部复制到 log buffer 中。我们现在假设有两个名为`T1`、`T2`的事务，每个事务都包含 2 个 mtr，我们给这几个 mtr 命名一下：

- 事务`T1`的两个`mtr`分别成为`mtr_T1_1`和`mtr_T1_2`
- 事务`T2`的两个`mtr`分别成为`mtr_T2_1`和`mtr_T2_2`

每个 mtr 都会产生一组 redo 日志，用示意图来描述一下这些 mtr 产生的日志情况：

<img src="img\image-20220223220152077.png" alt="image-20220223220152077" style="zoom:67%;" />


不同的事务可能是`并发`执行的，所以`T1`、`T2`之间的`mtr`可能是`交替执行`的。每当一个 mtr 执行完成时，伴随该 mtr 生成的一组 redo 日志就需要被复制到 log buffer 中，也就是说不同事务的 mtr 可能是交替写入 log buffer 的。

<img src="img\image-20220223220203041.png" alt="image-20220223220203041" style="zoom:67%;" />

有的 mtr 产生的 redo 日志量非常大，比如`mtr_t1_2`产生的 redo 日志占用空间比较大，占用了 3 个 block 来存储。

#### 3.redo log block的结构图

一个 redo log block 是由`日志头`、`日志体`、`日志尾`组成。日志头占用 12 字节，日志尾占用 8 字节，所以一个 block 真正能存储的数据就是 512-12-8=492 字节。

> **为什么一个 block 设计成 512 字节？**
>
> 这个和磁盘的扇区有关，机械磁盘默认的扇区就是 512 字节，如果你要写入的数据大于 512 字节，那么要写入的扇区肯定不止一个，这时就要涉及到盘片的转动，找到下一个扇区，假设现在需要写入两个扇区 A 和 B，如果扇区 A 写入成功，而扇区 B 写入失败，那么就会出现`非原子性`的写入，而如果每次只写入和扇区大小一样的 512 字节，那么每次的写入都是原子性的。

<img src="img\image-20220223220213363.png" alt="image-20220223220213363" style="zoom: 50%;" />

真正的 redo 日志都是存储到占用 `496` 字节大小的 `log block body` 中，图中的 `log block header` 和 `log block trailer` 存储的是一些管理信息。

<img src="img\image-20220223220242973.png" alt="image-20220223220242973" style="zoom: 50%;" />

- `log block header` 的属性分别如下：
  - `LOG_BLOCK_HDR_NO`：log buffer 是由 log block 组成,在内部 log buffer 就好似一个数组,因此 LOG_BLOCK_HDR_NO 用来标记这个数组中的位置。其是递增并且循环使用的,占用 4 个字节,但是由于第一位用来判断是否是 flush bit，所以最大的值为2G
  - `LOG_BLOCK_HDR_DATA_LEN`：表示 block 中已经使用了多少字节，初始值为`12`（因为`log block body`从第 12 个字节处开始）。随着往 block 中写入的 redo 日志越来越多，本属性值也跟着增长。如果`log block body`已经被全部写满，那么本属性的值被设置为`512`
  - `LOG_BLOCK_FIRST_REC_GROUP`：一条 redo 日志也可以被称之为一条 redo 日志记录（redo log record），一个 mtr 会产生多条 redo 日志记录，这些 redo 日志记录被称之为一个 redo 日志记录组（redo log record group）。LOG_BLOCK_FIRST_REC_GROUP 就代表该 block 中第一个 mtr 生成的 redo 日志记录组的偏移量（其实也就是这个 block 里第一个 mtr 生成的第一条 redo 日志的偏移量）。如果该值的大小和 LOG_BLOCK_HDR_DATA_LEN 相同，则表示当前 log block 不包含新的日志
  - `LOG_BLOCK_CHECKPOINT_NO`：占用 4 个字节，表示该 log block 最后被写入时的`checkpoint`
- `log block trailer` 中属性的意思如下：
  - `LOG_BLOCK_CHECKSUM`：表示 block 的校验值，用于正确性校验（其值和 LOG_BLOCK_HDR_NO 相同）

### 1.8  redo log file

#### 1.相关参数设置

- `innodb_log_group_home_dir`：指定 redo log 文件组所在的路径，默认值为`./`，表示在数据库的数据目录下。MySQL 的默认数据目录（`var/lib/mysql`）下默认有两个名为`ib_logfile0`和`ib_logfile1`的文件，log buffer 中的日志默认情况下就是刷新到这两个磁盘文件中。此 redo 日志文件位置还可以修改

- `innodb_log_files_in_group`：指明 redo log file 的个数，命名方式如：ib_logfile0，iblogfile1…iblogfilen。默认 2 个，最大 100 个

  ```mysql
  mysql> show variables like 'innodb_log_files_in_group';
  +---------------------------+-------+
  | Variable_name       | Value |
  +---------------------------+-------+
  | innodb_log_files_in_group | 2   |
  +---------------------------+-------+
  #ib_logfile0
  #ib_logfile1
  ```

- `innodb_flush_log_at_trx_commit`：控制 redo log 刷新到磁盘的策略，默认为 1

- `innodb_log_file_size`：单个 redo log 文件设置大小，默认值为`48M`。最大值为 512G，注意最大值指的是整个 redo log 系列文件之和，即（innodb_log_files_in_group * innodb_log_file_size）不能大于最大值 512G

  ```mysql
  mysql> show variables like 'innodb_log_file_size';
  +----------------------+----------+
  | Variable_name    | Value  |
  +----------------------+----------+
  | innodb_log_file_size | 50331648 |
  +----------------------+----------+
  ```

  根据业务修改其大小，以便容纳较大的事务。编辑 my.cnf 文件并重启数据库生效，如下所示

  ```mysql
  [root@localhost ~]# vim /etc/my.cnf
  innodb_log_file_size=200M
  ```

  > 在数据库实例更新比较频繁的情况下，可以适当加大 redo log 组数和大小，但不推荐设置过大，在 MySQL 崩溃恢复时会重新执行 REDO 日志中的记录

#### 2.日志文件组

从上边的描述中可以看到，磁盘上的`redo`日志文件不止一个，而是以一个`日志文件组`的形式出现。这些文件以`ib_logfile[数字]`（`数字`可以是`0`、`1`、`2`...）的形式进行命名，每个 redo 日志文件大小都是一样的。

在将 redo 日志写入日志文件组时，是从`ib_logfile0`开始写，如果`ib_logfile0`写满了，就接着`ib_logfile1`写，以此类推。如果写到最后一个文件，那就重新到`ib_logfile0`继续写，整个过程如下图所示：

<img src="img\image-20220223220325242.png" alt="image-20220223220325242" style="zoom: 50%;" />

总共的 redo 日志文件大小其实就是：`innodb_log_file_size × innodb_log_files_in_group`。

采用循环使用的方式向 redo 日志文件组里写数据的话，会导致后写入的 redo 日志覆盖掉前边写的 redo 日志？当然！所以 InnoDB 的设计者提出了 checkpoint 的概念。

#### 3.checkpoint

在整个日志文件组中还有两个重要的属性，分别是 write pos、checkpoint

- `write pos`是当前记录的位置，一边写一边后移
- `checkpoint`是当前要擦除的位置，也是往后推移

每次刷盘 redo log 记录到日志文件组中，write pos 位置就会后移。每次 MySQL 加载日志文件组恢复数据时，会清空加载过的 redo log 记录，并把 checkpoint 后移。write pos 和 checkpoint 之间的空间的部分可以用来写入新的 redo log 记录。

<img src="img\image-20220223220352164.png" alt="image-20220223220352164" style="zoom: 50%;" />



如果 write pos 追上 checkpoint ，表示**日志文件组满**了，这时候不能再写入新的 redo log 记录，MySQL 得停下来，清空一些记录，把 checkpoint 推进一下。

<img src="img\image-20220223220409647.png" alt="image-20220223220409647" style="zoom: 50%;" />



### 1.9 redo log 小结

InnoDB 的更新操作采用的是`Write Ahead Log`（预先日志持久化）策略，即先写日志，再写入磁盘。

<img src="img\image-20220310230526058.png" alt="image-20220310230526058" style="zoom: 67%;" />

redo 日志会携带一个`LSN（Log Sequence Number）`，同时每个数据页上也会记录一个 LSN，这个日志序列号可以用于数据页是否是脏页的判断

## 2.undo日志

redo log 是事务持久性的保证，undo log 是事务原子性的保证。在事务中`更新数据`的`前置操作`其实是要先写入一个 `undo log`。

### 2.1 如何理解undo日志

事务需要保证 `原子性`，也就是事务中的操作要么全部完成，要么什么也不做。但有时候事务执行到一半会出现一些情况：

- 事务执行过程中可能遇到各种错误，比如`服务器本身的错误`，`操作系统错误`，甚至是突然`断电`导致的错误
- 程序员可以在事务执行过程中手动输入 `ROLLBACK` 语句结束当前事务的执行

以上情况出现时，我们需要把数据改回原来的样子，这个过程称之为`回滚`，这样就可以造成一个假象：这个事务看起来什么都没做，符合`原子性`要求。

每当我们要对一条记录做改动时（可以是`INSERT`，`DELETE`，`UPDATE`），都需要把回滚时需要的东西记录下来，比如：

- `插入一条记录`时，至少要把这条记录的主键值记下来，之后回滚的时候只需要把这个主键值对应的`记录删除`。（对于每个 INSERT，InnoDB 存储引擎会完成一个 DELETE）
- `删除一条记录`时，至少要把这条记录中的内容都记下来，这样之后回滚时再把这些内容`插入`到表中（对于每个 DELETE，InnoDB 存储引擎会执行一个 INSERT）
- `修改了一条记录`，至少要把修改这条记录前的旧值都记录下来，这样回滚时再把这条记录`更新为旧值`。（对于每个 UPDATE，InnoDB 存储引擎会执行一个相反的 UPDATE，将修改前的数据恢复）

MySQL 把这些为了回滚而记录的内容称之为`撤销日志`或者`回滚日志`。注意，由于查询操作不会修改什么记录，所以在查询操作执行时，不会记录相应的 undo 日志。

此外，`undo log 会产生 redo log`，也就是 undo log 的产生会伴随着 redo log 的产生，因为 undo log 也需要持久性的保护。

### 2.2 undo日志的作用

- **作用1：回滚数据**

  用户对 undo 日志可能有误解：undo 用于将数据库物理地恢复到执行语句或事务之前的样子，但事实并非如此。undo 是`逻辑日志`，因此只是将数据库逻辑地恢复到原来的样子。所有修改都被逻辑地取消了，但是数据结构和页本身在回滚之后可能大不相同。

  这是因为在多用户并发系统中，可能会有数十、数百甚至数千个并发事务。数据库的主要任务就是协调对数据记录的并发访问。比如，一个事务在修改当前一个页中某几条记录，同时还有别的事务在对同一个页中另几条记录进行修改。因此，不能将一个页回滚到事务开始的样子，因为这样会影响其他事务正在进行的工作。

- **作用2：MVCC**

  undo 的另一个作用是 MVCC，即在 innoDB 存储引擎中 MVCC 的实现是通过 undo 来完成。当用户读取一行记录时，若该记录已经被其他事务占用，当前事务可以通过 undo 读取之前的行版本信息，以此实现非锁定读取。

### 2.3 undo的存储结构

#### 1.回滚段与undo页

InnoDB 对 undo log 的管理采用段的方式，也就是`回滚段（rollback segment）`。每个回滚段记录了 `1024` 个 `undo log segment`，而在每个 undo log segment 段中进行 `undo页`的申请。

- 在 `InnoDB1.1 版本之前`（不包括 1.1 版本），只有一个 rollback segment，因此支持同时在线的事务限制为 1024 。虽然对绝大多数的应用来说都已经够用
- 从 1.1 版本开始 InnoDB 支持最大 `128 个 rollback segment`，故其支持同时在线的事务限制提高到了 `128*1024`

```sql
mysql> show variables like 'innodb_undo_logs';
+------------------+-------+
| Variable_name  | Value |
+------------------+-------+
| innodb_undo_logs | 128  |
+------------------+-------+
```

虽然 InnoDB1.1 版本支持了 128 个 rollback segment，但是这些 rollback segment 都存储于共享表空间 ibdata 中。从 InnoDB1.2 版本开始，可通过参数对 rollback segment 做进一步的设置，这些参数包括：

- `innodb_undo_directory`：设置 rollback segment 文件所在的路径。这意味着 rollback segmen 可以存放在共享表空间以外的位置，即可以设置为独立表空间。该参数的默认值为`./`，表示当前 InnoDB 存储引擎的目录
- `innodb_undo_logs`：设置 rollback segment 的个数，默认值为 128。在 InnoDB1.2 版本中，该参数用来替换之前版本的参数 `innodb_rollback_segments`
- `innodb_undo_tablespaces`：设置构成 rollback segment 文件的数量，这样 rollback segment 可以较为平均地分布在多个文件中。设置该参数后，会在路径 `innodb_undo_directory` 看到 undo 为前缀的文件，该文件就代表 rollback segment 文件。默认值为 2

> undo log 相关参数一般很少改动。

**undo页的重用**

当我们开启一个事务需要写 undo log 的时候，就需要先去 undo log segment 中去找到一个空闲的位置，当有空位的时候，就去申请 undo 页，在这个申请到的 undo 页中进行 undo log 的写入。我们知道 mySQL 默认一页的大小是 16K。

为每一个事务分配一个页是非常浪费的（除非你的事务非常长），假设你的应用的 TPS（每秒处理的事务数目）为 1000，那么 1s 就需要 1000 个页，大概需要 16M 的存储，1 分钟大概需要 1G 的存储。如果照这样下去除非 MySQL 清理的非常勤快，否则随着时间的推移，磁盘空间会增长的非常快，而且很多空间都是浪费的。

于是 undo 页就被设计的可以`重用`了，当事务提交时，并不会立刻删除 undo 页。因为重用，所以这个 undo 页可能混杂着其他事务的 undo log。undo log 在 commit 后，会被放到一个`链表`中，然后判断 undo 页的使用空间是否`小于3/4`，如果小于 3/4 的话，则表示当前的 undo 页可以被重用，那么他就不会被回收，其他事务的 undo log 可以记录在当前 undo 页的后面。由于 undo log 是`离散的`，所以清理对应的磁盘空间时，效率不高。

#### 2.回滚段与事务

- 每个事务只会使用一个回滚段，一个回滚段在同一时刻可能会服务于多个事务
- 当一个事务开始的时候，会制定一个回滚段，在事务进行的过程中，当数据被修改时，原始的数据会被复制到回滚段
- 在回滚段中，事务会不断填充盘区，直到事务结束或所有的空间被用完。如果当前的盘区不够用，事务会在段中请求扩展下一个盘区，如果所有已分配的盘区都被用完，事务会覆盖最初的盘区或者在回滚段允许的情况下扩展新的盘区来使用
- 回滚段存在于 undo 表空间中，在数据库中可以存在多个 undo 表空间，但同一时刻只能使用一个 undo 表空间
- 当事务提交时，InnoDB 存储引擎会做以下两件事情：
  - 将 undo log 放入链表中，以供之后的 purge 操作
  - 判断 undo log 所在的页是否可以重用，若可以分配给下个事务使用

#### 3.回滚段中的数据分类

- `未提交的回滚数据(uncommitted undo information)`

  关联的时候并未提交，用于实现读一致性，所以该数据不能被其他事务的数据覆盖。

- `已经提交但未过期的回滚数据(committed undo information)`

  该数据关联的事务已经提交，但是仍然受到 undo retention 参数的保持时间的影响

- `事务已经提交并过期的数据(expired undo information)`

  事务已经提交，而且数据保存时间已经超过 undo rentention 参数指定的时间，属于已经过期的数据。当回滚段满了之后，会优先`覆盖事务已经提交并且过期`的数据。

事务提交后并不能马上删除 undo log 以及 undo log 所在的页。这是因为可能还有其他事务需要通过 undo log 来得到行记录之前的版本。故事务提交时将 undo log 放入一个链表中，是否可以删除 undo log 及 undo log 所在的页由 purge 线程来判断。

### 2.4 undo的类型

在 InnoDB 存储引擎中，undo log 分为：

- `insert undo log`

  insert undo log 是指在 insert 操作中产生的 undo log。因为 insert 操作纪律，只对事务本身可见，对其他事务不可见。故该 undo log 可以在事务提交后直接删除。`不需要进行 purge 操作`。

- `update undo log`

  update undo log 记录的是对 delete 和 update 操作产生的 undo log。该 undo log 可能需要提供 MVCC 机制，因此不能再事务提交时就进行删除。提交时放入 undo log 链表，等待 purge 线程进行最后的删除。

### 2.5 undo log的生命周期

#### 1.简要生成过程

以下是 undo + redo 事务的简化过程：

假设有 2 个数值，分别为 A=1 和 B=2，然后将 A 修改为 3，B 修改为 4

```sql
1. start transaction;
2. 记录 A=1 到 undo log;
3. update A=3;
4. 记录 A=3 到 redo log;
5. 记录 B=2 到 undo log；
6. update B=4;
7. 记录 B=4 到 redo log；
8. 将 redo log 刷新到磁盘
9. commit
```

- 在 1-8 步骤的任意一步系统宕机，事务未提交，该事务就不会对磁盘上的数据做任何影响
- 如果在 8-9 之间宕机，恢复之后可以选择回滚，也可以选择继续完成事务提交，因为此时 redo log 已经持久化
- 若在 9 之后系统宕机，内存映射中变更的数据还来不及刷回磁盘，那么系统恢复后，可以根据 redo log 把数据刷回磁盘

**只有Buffer Pool的流程：**

<img src="img\image-20220223220510612.png" alt="image-20220223220510612" style="zoom: 67%;" />

**有了Redo Log和Undo Log之后：**

<img src="img\image-20220223220520969.png" alt="image-20220223220520969" style="zoom:67%;" />

简述：先找是否有加载对应的 BufferPool 有就使用没有就加载读取 -> 写入UndoLog -> 操作数据 -> 写入 RedoLogBuffer 内存 -> 写入 RedoLog 到磁盘文件

#### 2.详细生成过程

对于 InnoDB 引擎来说，每个[行记录](./第07章_InnoDB数据存储结构.mmap)除了记录本身的数据之外，还有几个隐藏的列：

- `DB_ROW_ID`：如果没有为表显示的定义主键，并且表中也没有定义唯一索引，那么 InnoDB 会自动为表添加一个 row_id 的隐藏列作为主键
- `DB_TRX_ID`：每个事务都会分配一个事务 ID，当对某条记录发生变更时，就会将这个事务的事务 ID 写入 trx_id 中
- `DB_ROLL_PTR`：回滚指针，本质上就是 undo log 的指针

<img src="img\image-20220223220533745.png" alt="image-20220223220533745" style="zoom:67%;" />

**当我们执行INSERT时：**

```sql
begin;
INSERT INTO user (name) VALUES ("tom");
```

插入的数据都会生成一条 insert undo log，并且数据的回滚指针会指向它。undo log 会记录 undo log 的序号、插入主键的列和值等，那么在进行回滚的时候，通过主键直接把对应的数据删除即可。

<img src="img\image-20220223220550950.png" alt="image-20220223220550950" style="zoom:67%;" />

**当我们执行UPDATE时：**

对于更新的操作会产生 update undo log，并且会分更新主键和不更新主键的，假设现在执行：

```sql
UPDATE user SET name="Sun" WHERE id=1;
```

<img src="img\image-20220223220606362.png" alt="image-20220223220606362" style="zoom:67%;" />

这时会把老的记录写入新的 undo log，让回滚指针指向新的 undo log，它的 undo No 是 1，并且新的 undo log 会指向老的 undo log（undo No 是 0）。

假设现在执行：

```sql
UPDATE user SET id=2 WHERE id=1;
```

<img src="img\image-20220223220624584.png" alt="image-20220223220624584" style="zoom:80%;" />

对于更新主键的操作，会先把原来的数据 deletemark 标识打开，这时并没有真正的删除数据，真正的删除会交给清理线程去判断，然后在后面插入一条新的数据，新的数据也会产生 undo log，并且 undo log 的序号会递增。

可以发现每次对数据的变更都会产生一个 undo log，当一条记录被变更多次时，就会产生多条 undo log，undo log 记录的是变更前的日志，并且每个 undo log 的序号是递增的，当要回滚的时候，`按照序号依次向前推`，就可以找到原始的数据。

#### 3.undo log是如何回滚的

以上面的例子来说，假设执行 rollback，那么对应的流程应该是这样：
- 通过 undo no=3 的日志把 id=2 的数据删除
- 通过 undo no=2 的日志把 id=1 的数据的 deletemark 还原成 0
- 通过 undo no=1 的日志把 id=1 的数据的 name 还原成 Tom
- 通过 undo no=0 的日志把 id=1 的数据删除

#### 4.undo log的删除

- 针对于`insert undo log`

  因为 insert 操作的记录，只对事务本身可见，对其他事务不可见。故该 undo log 可以在事务提交后`直接删除`，不需要进行 purge 操作。

- 针对于`update undo log`

  该 undo log 可能需要提供 MVCC 机制，因此不能在事务提交时就进行删除。提交时放入 undo log 链表，等待 purge 线程进行最后的删除。

> 补充：
>
> purge 线程两个主要作用是：`清理 undo 页`和`清除 page 里面带有 Delete_Bit 标识的数据行`。在 InnoDB 中，事务中的 Delete 操作实际上并不是真正的删除掉数据行，而是一种 Delete mark 操作，在记录上标识 Delete_Bit，而不删除记录，真正的删除工作需要后台 purge 线程去完成。.

### 2.6 小结

<img src="img\image-20220223220701657.png" alt="image-20220223220701657" style="zoom: 50%;" />

`undo log`是逻辑日志，对事务回滚时，只是将数据库逻辑地恢复到原来的样子。

`redo log`是物理日志，记录的是数据页的物理变化，undo log 不是 redo log 的逆过程（逻辑上的逆过程，但是物理上不是直接的逆过程）。
