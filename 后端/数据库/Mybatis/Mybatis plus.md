# Mybatis plus

[github](https://github.com/baomidou/mybatis-plus)

[文档](https://baomidou.com/pages/24112f/#特性)

## 1. 快速开始

> - 只需要创建 mapper 接口并继承 BaseMapper 接口，不需要创建 sql 映射文件，就可以使用所有 CRUD 操作
> - 在 BaseMapper 中已经提供了各种基础 CRUD 方法
>
> ```properties
> # mysql 8 的驱动，可向下兼容
> spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
> spring.datasource.username=root
> spring.datasource.password=root
> # mysql 8 需要增加时区配置
> spring.datasource.url=jdbc:mysql://localhost:3306/mybatis? useSSL=false & userUnicode=true & characterEncoding=utf-8 & serverTimezone=GMT%2B8
> ```
>
> ```java
> // 在对应的 mapper 上继承 BaseMapper 接口
> // 泛型指定操作的实例类
> @Mapper
> public interface UserMapper extends BaseMapper<User> {
> }
> ```

## 2. 配置日志

```properties
mybatis-plus.configuration.log-impl=org.apache.ibatis.logging.stdout.StdOutImpl
```

## 3. 插入测试及主键生成策略

> - 插入时 ID 会自动生成
> - 数据库中的主键生成策略（uuid，自增id，雪花算法，redis，zookeeper）
>   - [参考](https://www.cnblogs.com/haoxinyue/p/5208136.html)
> - 雪花算法：
>   - snowflake是Twitter开源的分布式ID生成算法，结果是一个long型的ID
>   - 其核心思想是：使用41bit作为毫秒数，10bit作为机器的ID（5个bit是数据中心：北京，上海等，5个bit的机器ID），12bit作为毫秒内的流水号（意味着每个节点在每毫秒可以产生 4096 个 ID），最后还有一个符号位，永远是0
>   - 具体实现的代码可以参看https://github.com/twitter/snowflake。雪花算法支持的TPS可以达到419万左右（2^22*1000）
> - `@TableId(type=IdType.AUTO)`：主键自增
> - `@TableId(type=IdType.NONE)`：未设置主键
> - `@TableId(type=IdType.INPUT)`：：手动输入，插入时不指定 id 则为 null，需要自己配置
> - `@TableId(type=IdType.ID_WORKER)`：默认的全局唯一 id，使用雪花算法
> - `@TableId(type=IdType.INPUT)`：全局唯一 id，uuid
> - `@TableId(type=IdType.INPUT)`：IDWORKER 字符串表示法

## 4. 更新操作

> `userMapper.updateById(user)`：所有的 sql 都会被自动动态配置，只需传入对象即可

### 4.1 自动填充

> 所有的数据库表：**gmt_create**，**gmt_modified** 几乎所有的表都要配置上，而且需要自动化
>
> 1. 数据库级别(不推荐)
>
>    在表中新增字段：create_time，update_time，添加默认值
>
> 2. 代码级别
>
>    1. 删除数据库的默认值
>
>    2. 在实体类的字段属性上添加注解：`@TableField`
>
>       ```java
>       // 添加填充时机
>       @TableField(fill= FieldFill.INSERT)
>       private Date createTime;
>       @TableField(fill= FieldFill.INSERT_UPDATE)
>       private Date updateTime;
>       ```
>
>       > **扩展：TypeHandler**
>       >
>       > ```java
>       > @Data
>       > @AllArgsConstructor
>       > // autoResultMap=true 指定自动 map
>       > @TanbleName(value="customer", autoResultMap=true)
>       > public class Customer {
>       >     @TableId(type=IdType.AUTO)
>       >     private Integer id;
>       >     // 指定 typeHandler
>       >     @TanbleField(typeHandler=AESTypeHandler.class)
>       >     private String email;
>       > }
>       > ```
>       >
>       > 这种方法仅适用于 plus 提供的默认方法，如果自定义查询方法则需要按照[自定义类型处理器](Mybatis 3.md#10.4 自定义类型处理器)在 XML 中配置。
>    
>    3. 自定义 MetaObjectHandler 的实现类处理注解（3.0.5 version 的方法，最新版见官网）
>    
>       ```java
>       /**
>       * 旧版
>       */
>       @Component
>       public class MyMetaObjectHandler implements MetaObjectHandler {
>       
>           // 插入时的填充策略
>           @Override
>           public void insertFill(MetaObject metaObject) {
>               this.setFieldValByName("createTime", new Date(), metaObject);
>               this.setFieldValByName("updateTime", new Date(), metaObject);
>           }
>       
>           // 更新时的填充策略
>           @Override
>           public void updateFill(MetaObject metaObject) {
>               this.setFieldValByName("updateTime", new Date(), metaObject);
>           }
>       }
>       
>       /**
>       * 新版
>       */
>       @Component
>       public class MyMetaObjectHandler implements MetaObjectHandler {
>       
>           @Override
>           public void insertFill(MetaObject metaObject) {
>               log.info("start insert fill ....");
>               this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now()); // 起始版本 3.3.0(推荐使用)
>               // 或者
>               this.strictInsertFill(metaObject, "createTime", () -> LocalDateTime.now(), LocalDateTime.class); // 起始版本 3.3.3(推荐)
>               // 或者
>               this.fillStrategy(metaObject, "createTime", LocalDateTime.now()); // 也可以使用(3.3.0 该方法有bug)
>           }
>       
>           @Override
>           public void updateFill(MetaObject metaObject) {
>               log.info("start update fill ....");
>               this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now()); // 起始版本 3.3.0(推荐)
>               // 或者
>               this.strictUpdateFill(metaObject, "updateTime", () -> LocalDateTime.now(), LocalDateTime.class); // 起始版本 3.3.3(推荐)
>               // 或者
>               this.fillStrategy(metaObject, "updateTime", LocalDateTime.now()); // 也可以使用(3.3.0 该方法有bug)
>           }
>       }
>       ```

### 4.2 乐观锁

> - 乐观锁：认为不会出现问题，无论干什么都不会上锁，如果出现了问题，再次更新值操作
>
> - 悲观锁：认为总是会出现问题，无论干什么都会上锁后再去操作
>
> - 乐观锁实现方式：
>
>   1. 取出记录时，获取当前 **version**
>   2. 更新时，带上这个 version
>   3. 执行更新时， set version = newVersion where version = oldVersion
>   4. 如果 version 不对，就更新失败
>
> - 实现步骤：
>
>   1. 实体类中对应字段上添加注解：`@Version`
>
>      ```java
>       @Version
>      private Integer version;
>      ```
>
>   2. 注册组件（最新版方式不同）
>
>      ```java
>      @Configuration
>      @MapperScan("按需修改")
>      public class MybatisPlusConfig {
>          /**
>           * 旧版
>           */
>          @Bean
>          public OptimisticLockerInterceptor optimisticLockerInterceptor() {
>              return new OptimisticLockerInterceptor();
>          }
>                     
>          /**
>           * 新版
>           */
>          @Bean
>          public MybatisPlusInterceptor mybatisPlusInterceptor() {
>              MybatisPlusInterceptor mybatisPlusInterceptor = new MybatisPlusInterceptor();
>              mybatisPlusInterceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
>              return mybatisPlusInterceptor;
>          }
>      }
>      ```

## 5. 查询操作

> 1. 基础查询操作
>
>    ```java
>    @Test
>    void testSelect() {
>        // 基础查询
>        User user = userMapper.selectById(1L);
>        // 批量查询
>        List<User> users = userMapper.selectBatchIds(Arrays.asList(1L, 2L, 3L));
>        // 条件查询
>        HashMap<String, Object> map = new HashMap<>();
>        // 自定义查询条件
>        map.put("name", "Tom");
>        userMapper.selectByMap(map);
>    }
>    ```
>
> 2. 分页查询
>
>    1. 原始的 limit 分页查询
>
>    2. pageHelper 等第三方插件
>
>    3. MP 自带的分页插件
>
>       1. 配置拦截器
>
>          ```java
>          @Configuration
>          @MapperScan("com.baomidou.cloud.service.*.mapper*")
>          public class MybatisPlusConfig {
>          
>              // 旧版
>              @Bean
>              public PaginationInterceptor paginationInterceptor() {
>                  return new PaginationInterceptor();
>              }
>          
>              // 最新版
>              @Bean
>              public MybatisPlusInterceptor mybatisPlusInterceptor() {
>                  MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
>                  interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.H2));
>                  return interceptor;
>              }
>          
>          }
>          ```
>
>       2. 测试
>
>          ```java
>          @Test
>          void testPage() {
>              Page<User> page = new Page<>(2, 5);
>              userMapper.selectPage(page, null);
>              List<User> records = page.getRecords();
>              page.hasNext();
>              page.hasPrevious();
>              page.getTotal();
>          }
>          ```

## 6. 删除操作

> 1. 基本删除操作
>
>    ```java
>    @Test
>    void testDelete(){
>        userMapper.deleteById(1477220741467344898L);
>        userMapper.deleteBatchIds(Arrays.asList(1477220741467344899L));
>        userMapper.deleteByMap(new HashMap(){{put("id","1477220741467344900");put("id","5");}});
>    }
>    ```
>
> 2. 逻辑删除
>
>    - 在数据库中没有被移除，而是通过变量表示失效的状态
>
>    - 在实体类的相应字段上添加注解：`@TableLogic`
>
>      ```java
>      // 逻辑删除
>      @TableLogic
>      private Integer deleted;
>      ```
>
>    - 配置
>
>      - 老版本
>
>        ```java
>        @Bean
>        public ISqlInjector iSqlInjector() {
>            return new LogicSqlInjector();
>        }
>        ```
>
>      - 新版本
>
>        ```yaml
>        mybatis-plus:
>          global-config:
>            db-config:
>              logic-delete-field: flag # 全局逻辑删除的实体字段名(since 3.3.0,配置后可以忽略不配置步骤2)
>              logic-delete-value: 1 # 逻辑已删除值(默认为 1)
>              logic-not-delete-value: 0 # 逻辑未删除值(默认为 0) 
>        ```
>
>    - 测试
>
>      ```java
>      @Test
>      void testLogicDelete() {
>          // 删除时执行的是更新操作（更新 逻辑删除字段）
>          int i = userMapper.deleteById(1477220741467344900L);
>          // 查询时会自动过滤被删除的字段（and deleted=0）
>          User user = userMapper.selectById(1477220741467344900L);
>          System.out.println(user);　// null
>      }
>      ```

## 7. 性能分析插件

> MP 提供性能分析插件，如果超过这个时间就停止执行，但该插件有性能损耗，不建议生产环境使用。
>
> - Maven 引入
>
>   ```xml
>   <!-- https://mvnrepository.com/artifact/p6spy/p6spy -->
>   <dependency>
>       <groupId>p6spy</groupId>
>       <artifactId>p6spy</artifactId>
>       <version>3.9.1</version>
>   </dependency>
>   ```
>
> - application.yml 配置：
>
>   ```properties
>   spring.datasource.driver-class-name=com.p6spy.engine.spy.P6SpyDriver
>   spring.datasource.username=root
>   spring.datasource.password=root
>   spring.datasource.url=jdbc:p6spy:mysql://localhost:3306/mybatis?useSSL=false&userUnicode=true&characterEncoding=utf-8&serverTimezone=GMT%2B8
>   ```
>
> - spy.properties 配置：
>
>   ```properties
>   #3.2.1以上使用
>   modulelist=com.baomidou.mybatisplus.extension.p6spy.MybatisPlusLogFactory,com.p6spy.engine.outage.P6OutageFactory
>   #3.2.1以下使用或者不配置
>   #modulelist=com.p6spy.engine.logging.P6LogFactory,com.p6spy.engine.outage.P6OutageFactory
>   # 自定义日志打印
>   logMessageFormat=com.baomidou.mybatisplus.extension.p6spy.P6SpyLogger
>   #日志输出到控制台
>   appender=com.baomidou.mybatisplus.extension.p6spy.StdoutLogger
>   # 使用日志系统记录 sql
>   #appender=com.p6spy.engine.spy.appender.Slf4JLogger
>   # 设置 p6spy driver 代理
>   deregisterdrivers=true
>   # 取消JDBC URL前缀
>   useprefix=true
>   # 配置记录 Log 例外,可去掉的结果集有error,info,batch,debug,statement,commit,rollback,result,resultset.
>   excludecategories=info,debug,result,commit,resultset
>   # 日期格式
>   dateformat=yyyy-MM-dd HH:mm:ss
>   # 实际驱动可多个
>   #driverlist=org.h2.Driver
>   # 是否开启慢SQL记录
>   outagedetection=true
>   # 慢SQL记录标准 2 秒
>   outagedetectioninterval=2
>   ```

## 8. 条件构造器

> 用来构造复杂的 sql 语句
>
> ```java
> @Test
> void testWrapper() {
>     // name 不为空并且邮箱不为空，并且年龄大于等于21
>     QueryWrapper<User> wrapper = new QueryWrapper<>();
>     wrapper
>         .isNotNull("name")
>         .isNotNull("email")
>         .ge("age", 21);
>     userMapper.selectList(wrapper).forEach(System.out::println);
> }
> ```
>
> ```java
> @Test
> void testWrapper() {
>     // 名字等于 Tom
>     QueryWrapper<User> wrapper = new QueryWrapper<>();
>     wrapper.eq("name", "Tom");
>     userMapper.selectOne(wrapper);
> }
> ```
>
> ```java
> @Test
> void testWrapper() {
>     // 年龄在 20 ~ 25 之间的用户数
>     QueryWrapper<User> wrapper = new QueryWrapper<>();
>     wrapper.between("age", 20, 25);
>     userMapper.selectCount(wrapper);
> }
> ```
>
> ```java
> @Test
> void testWrapper() {
>     // 名字不包含 o
>     QueryWrapper<User> wrapper = new QueryWrapper<>();
>     wrapper
>         // %o%
>         .notLike("name", "o")
>         // t%
>         .likeRight("email", "t");
>     userMapper.selectMaps(wrapper).forEach(System.out::println);
> }
> ```
>
> ```java
> @Test
> void testWrapper() {
>     QueryWrapper<User> wrapper = new QueryWrapper<>();
>     // id 在子查询中查出
>     // WHERE (id IN (select id from user where id<3))
>     wrapper.inSql("id", "select id from user where id<3");
>     userMapper.selectObjs(wrapper).forEach(System.out::println);
> }
> ```
>
> ```java
> @Test
> void testWrapper() {
>     QueryWrapper<User> wrapper = new QueryWrapper<>();
>     // 通过 id 进行排序
>     wrapper.orderByDesc("id");
>     userMapper.selectList(wrapper).forEach(System.out::println);
> }
> ```

## 9. 代码生成器

### 1. 引入

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-generator</artifactId>
    <version>3.5.1</version>
</dependency>
<dependency>
    <groupId>org.freemarker</groupId>
    <artifactId>freemarker</artifactId>
    <version>2.3.28</version>
    <scope>compile</scope>
</dependency>
```

### 2. 快速生成

[参考](https://baomidou.com/pages/981406/#数据库配置-datasourceconfig)

```java
public class CodeGenerator {
    public static void main(String[] args) {
        String property = System.getProperty("user.dir");
        FastAutoGenerator.create("jdbc:mysql://localhost:3306/mybatis?useSSL=false&userUnicode=true&characterEncoding=utf-8&serverTimezone=GMT%2B8"
                        , "root"
                        , "root")
                .globalConfig(builder -> {
                    builder.author("youyi zhao") // 设置作者
                            .enableSwagger() // 开启 swagger 模式
                            .fileOverride() // 覆盖已生成文件
                            .dateType(DateType.TIME_PACK) // 日期格式
                            .outputDir(property + "/src/main/java"); // 指定输出目录
                })
                .packageConfig(builder -> {
                    builder.parent("com.example.mybatisplus") // 设置父包名
                            .moduleName("codeGeneration") // 设置父包模块名
                            .pathInfo(Collections.singletonMap(OutputFile.mapperXml, property + "/src/main/java/mapper")); // 设置mapperXml生成路径
                })
                .strategyConfig(builder -> {
                    builder.addInclude("user") // 设置需要生成的表名
                            .addTablePrefix("tbl_", "c_") // 设置过滤表前缀
                            // 实体类策略
                            .entityBuilder()
                            .versionColumnName("version") // 乐观锁
                            .logicDeleteColumnName("deleted") // 逻辑删除
                            // 自动填充配置
                            .addTableFills(new Column("create_time", FieldFill.INSERT))
                            .addTableFills(new Property("updateTime", FieldFill.INSERT_UPDATE))
                            .idType(IdType.AUTO)
                            // Controller 策略
                            .controllerBuilder()
                            .enableHyphenStyle()
                            .enableRestStyle();
                })
                .templateEngine(new FreemarkerTemplateEngine()) // 使用Freemarker引擎模板，默认的是Velocity引擎模板
                .execute();
    }
}
```

### 3. 交互式生成

```java
FastAutoGenerator.create(DATA_SOURCE_CONFIG)
    // 全局配置
    .globalConfig((scanner, builder) -> builder.author(scanner.apply("请输入作者名称？")).fileOverride())
    // 包配置
    .packageConfig((scanner, builder) -> builder.parent(scanner.apply("请输入包名？")))
    // 策略配置
    .strategyConfig((scanner, builder) -> builder.addInclude(getTables(scanner.apply("请输入表名，多个英文逗号分隔？所有输入 all")))
                        .controllerBuilder().enableRestStyle().enableHyphenStyle()
                        .entityBuilder().enableLombok().addTableFills(
                                new Column("create_time", FieldFill.INSERT)
                        ).build())
    /*
        模板引擎配置，默认 Velocity 可选模板引擎 Beetl 或 Freemarker
       .templateEngine(new BeetlTemplateEngine())
       .templateEngine(new FreemarkerTemplateEngine())
     */
    .execute();


// 处理 all 情况
protected static List<String> getTables(String tables) {
    return "all".equals(tables) ? Collections.emptyList() : Arrays.asList(tables.split(","));
}
```

