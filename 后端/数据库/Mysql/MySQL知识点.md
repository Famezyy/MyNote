# MySQL知识点

## 1.NULL注意点

### **1. count 数据丢失**

> - 当某列存在 NULL 值时，使用 `count(colum)` 查询该列，就会出现数据丢失问题
> - ==解决==：若某列存在 NULL 值时，使用 `count(*)` 或者 `count(id)` 进行数据统计
>

### **2. distinct 数据丢失**

> - 当使用 `count(distinct col1, col2)` 查询时，若其中一列为 NULL，那么即使另一列有不同的值，那么查询结果也会丢失数据： 
>
>   `select count(distinct name, mobile) from person`
>

### **3. select 数据丢失**

> - 如果某列存在 NULL 值，如果执行非等于查询（<> / !=）则会导致 NULL 值的结果丢失：
>
>   `select * from person where name <> 'java' order by id`
>
> - ==解决==：`select * from person where name <> 'java' or isnull(name) order by id`
>

### **4. 聚合函数计算时不会考虑 null 值**

#### 导致空指针异常

> - 如果某列存在 NULL 值，可能会导致 sum(col) 的返回结果为 NULL 而非 0，如果 sum 查询的结果为 NULL 就可能会导致程序执行时空指针异常：
>
>   `select sum(num) from goods where id > 4`（当仅有一个 id 大于 4 的数据且该数据值为 null 时）
>
> - ==解决==：`select ifnull(sum(num), 0) from goods where id > 4`
>

### **5. 增加了查询难度**

> - 当进行 NULL 值查询时，必须使用 NULL 匹配的查询方法，比如 `IS NULL` 或者 `IS NOT NULL` 或者 `IFNULL(com)`，而不能使用传统的 =，!=，<>...
>   - `select * from person where name != null` 结果为空
>
> - 正确写法：
>   - `select * from person where name is not null`
>   - `select * from person where name !isnull(name)`：推荐，效率更快
>

### **6. NULL 不会影响索引**

### **7. NOT IN 返回 NULL**

> - 当给定的值不在列表中且列表中含有 null 时，结果返回 null
>
>   ```mysql
>   SELECT "2" NOT IN ("3", "1", NULL); # 会返回 null
>   SELECT "2" NOT IN ("3", "1", "4");  # 返回 1
>   SELECT "2" NOT IN ("3", "1", "2");  # 返回 0
>   ```

### 8.节约空间

1. 修改行格式

   Compressed 在发生行溢出时会以 Zlib 算法将溢出页压缩

2. 修改编码

   - UTF-8 占用 3 或 4 个字节
   - 全英文考虑使用 ASIIC

3. 使用 VARCHAR 或 CHAR 时声明字段大小

3. 声明为 NOT NULL，节省 1 字节

4. 整数避免使用字符串

5. 主键不要太长

   - 聚簇索引
   - 二级索引

7. 在服务层使用 map 映射通用的字段，数据库中只存储编号即可

6. 没有创建索引且没有 UNIQUE 字段时，默认会创建一个 row_id 作为索引，占 6 字节，可自己定义索引

### 9.使用函数时

对应列上的索引将失去作用

## 2.8.0 和 5.7 区别

- 默认字符集不同

  5.7 默认的客户端和服务器 latin1，utf8 字符集指向的是 utf8mb3，8.0 默认 utf8mb4

- 8.0 的 DDL 语句支持原子操作

- 整数数据类型

- 从 MySQL 8.0.17 开始，不推荐使用显示宽度属性

## 3.索引的特点

**优点**

- 通过`B+树`的结构来存储数据，可以大大减少数据检索时的`磁盘IO`次数，从而提升数据查询的性能
- `B+树`索引在进行范围查找的时候，只需要找到起始节点，然后基于叶子节点的链表结构往下读取即可查询，效率较高
- 通过唯一索引约束，可以保证数据表中每一行数据的唯一性

**缺点**

- 数据的增删改需要涉及到索引的维护，当数据量较大的情况下，索引的维护会带来较大的`性能开销`
- 一个表中允许存在一个聚簇索引和多个非聚簇索引，但是索引数不能创建太多，否则索引维护成本过高
- 创建索引的时候，需要考虑到索引字段值的分散性，如果字段的重复数据过多，创建索引反而会带来性能降低
- 若主键索引的长度过大，则由于不论是聚簇索引还是多个非聚簇索引，每个节点都会存储主键索引，因此会造成较大的空间开销

## 4.备份、删除、回撤

```sql
# 备份
CREATE TABLE mytable_2023_1_2 SELECT * FROM mytable;
# 删除
DELETE FROM mytable LIMIT 1;
# 回撤
INSERT INTO mytable SELECT * FROM mytable_2023_1_2;
```

**通过 EVENT 定时定量删除**

```sql
DROP EVENT event_delete;
DELIMITER $$
	CREATE EVENT IF NOT EXISTS event_delete
	ON SCHEDULE EVERY 1 SECOND on COMPLETION PRESERVE
	DO BEGIN
		DECLARE num integer;
		SELECT COUNT(*) INTO num FROM mytable;
		IF num > 0 
			THEN DELETE FROM mytable limit 1;
		END IF;
END$$
```

查看：`SHOW VARIABLES LIKE 'event_scheduler';`

开启：`SET GLOBAL event_scheduler = 1; `

关闭：`SET GLOBAL event_scheduler = 0;`

## 5.B 树和 B+ 树

- B 树叶子节点存储数据，所以存储的索引数量少，因此通常 B 树的深度比 B+ 树大，所需磁盘 IO 更多，查询效率低
- B 树查询不稳定，B+ 树数据都在叶子节点，每次查询访问的节点数相同，查询效率稳定
- 范围查询时，B 树需要通过中序遍历，B+ 树叶子节点间是双向链表结构，叶子节点内部是单向链表结构，只需通过指针便利即可

## 6.多线程读写数据

MySQL 默认有 4 条读线程和 4 条写线程，1 条 master 线程，1 条 page cleaner 线程，1 条 purge 线程。

**案例：读取 1 亿条数据插入数据库**

- 创建读取文件的线程和插入数据库的线程，中间用 MQ 衔接
  - 读取文件
    - 考虑服务器的内存设定一次读取的字节数
    - 不考虑插入顺序的话可以多线程读取
  - 插入数据库
    - 使用`batch`插入，减少与数据库的交互成本，提高插入效率
    - 插入线程可以创建 MySQL 最大的读线程数，最大效率利用 MySQL 线程；例如如果服务器 CPU 是 16 核，则可以分别指定 6 条读写线程，1 条 master 线程，1 条 page cleaner 线程，1 条 purge 线程
