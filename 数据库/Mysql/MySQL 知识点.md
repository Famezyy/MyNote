# mySQL

## 注意点

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

#### **4.1. 导致空指针异常**

> - 如果某列存在 NULL 值，可能会导致 sum(col) 的返回结果为 NULL 而非 0，如果 sum 查询的结果为 NULL 就可能会导致程序执行时空指针异常：
>
>   `select sum(num) from goods where id > 4`（当仅有一个 id 大于 4 的数据且该数据值为 null 时）
>
> - ==解决==：`select ifnull(sum(num), 0) from goods where id > 4`
>

### **5. 增加了查询难度**

> - 当进行 NULL 值查询时，必须使用 NULL 只匹配的查询方法，比如 `IS NULL` 或者 `IS NOT NULL` 或者 `IFNULL(com)`，而不能使用传统的 =，!=，<>...
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

### **8.节约空间**

1. 修改行格式

   Compressed 在发生行溢出时会以 Zlib 算法将溢出页压缩

2. 修改编码

   - UTF-8 占用 3 或 4 个字节
   - 全英文考虑使用 ASIIC

3. 使用 VARCHAR 或 CHAR 时声明字段大小

3. 声明为 NOT NULL，节省 1字节

4. 整数避免使用字符串

5. 主键不要太长

   - 聚簇索引
   - 二级索引

7. 在服务层使用 map 映射通用的字段，数据库中只存储编号即可

6. 没有创建索引且没有 UNIQUE 字段时，默认会创建一个 row_id 作为索引，占 6 字节，可自己定义索引

### **9.使用函数时**

1. 对应列上的索引将失去作用

## 8.0 和 5.7 区别

### **默认字符集不同**

5.7 默认的客户端和服务器 latin1，utf8 字符集指向的是 utf8mb3，8.0 默认 utf8mb4

### **8.0 的 DDL 语句支持原子操作**

### **整数数据类型**

从MySQL 8.0.17开始，不推荐使用显示宽度属性
