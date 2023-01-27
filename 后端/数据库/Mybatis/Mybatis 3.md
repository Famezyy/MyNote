# Mybatis 3

[TOC]

## 简介

<img src="..\img\image-20211217001005534.png" alt="image-20211217001005534" style="zoom: 67%;" />

<img src="..\img\image-20211217001238475.png" alt="image-20211217001238475" style="zoom:67%;" />

<img src="..\img\image-20211217001321246.png" alt="image-20211217001321246" style="zoom:67%;" />

## 1. 快速开始

### 1.1 无接口方式

#### 1.1.1 创建全局配置文件

```xml
mybatis-config.xml
===================================================================================
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/mybatis"/>
                <property name="username" value="root"/>
                <property name="password" value="root"/>
            </dataSource>
        </environment>
    </environments>
    <!-- 写好的 sql 映射文件注册到全局配置文件中 -->
    <mappers>
        <mapper resource="EmployeeMapper.xml"/>
    </mappers>
</configuration>
```

#### 1.1.2 创建 sql 映射文件

配置了每个 sql，以及 sql 的封装规则等。

```xml
EmployeeMapper.xml
============================================================
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.youyi.EmployeeMapper">
<!--
	namespace：命名空间
    id：唯一标识
    #{id}：从传过来的参数中获取值
-->
    <select id="getEmpById" resultType="bean.Employee">
        select id, last_name lastName, email, gender from tbl_employee where id = #{id}
    </select>
</mapper>
```

#### 1.1.3 创建相应的 bean 文件 Employee

#### 1.1.4 执行操作

```java
@Test
public void test01() throws IOException {
    String resource = "mybatis-config.xml";
    InputStream inputStream = Resources.getResourceAsStream(resource);
    // 1. 根据全局配置文件得到 sqlSessionFactory
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

    // 2. 获取sqlSession实例，能直接执行已经映射的sql语句
    SqlSession sqlSession = sqlSessionFactory.openSession();
    try {
        // 3. 使用 sql 的唯一标识告诉Mybatis执行哪个 sql，sql 都是保存在 sql映射文件中
        // 第一个参数：sql 的唯一标识（命名空间 + id）
        // 第二个参数：执行 sql 的参数
        Employee e = sqlSession.selectOne("com.youyi.EmployeeMapper.getEmpById", 1);
        System.out.println(e);
    } finally {
        // 4. 一个sqlSession代表和数据库的一次会话，用完关闭
        sqlSession.close();
    }
}
```

### 1.2 接口方式

#### 1.2.1 创建全局配置文件

#### 1.2.2 创建 mapper 接口

```java
public interface EmployeeMapper {
    public Employee getEmpById(Integer id);
}
```

#### 1.2.3 创建 sql 映射文件与接口映射

```xml
EmployeeMapper.xml
==============================================================================================================
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="dao.EmployeeMapper">
<!--
    namespace：指定为接口的全类名
    id：唯一标识，指定为接口的方法
    #{id}：从传过来的参数中获取值
-->
    <select id="getEmpById" resultType="bean.Employee">
        select id, last_name lastName, email, gender from tbl_employee where id = #{id}
    </select>
</mapper>
```

#### 1.2.4 执行

```java
@Test
public void test02() throws IOException {
    String resource = "mybatis-config.xml";
    InputStream inputStream = Resources.getResourceAsStream(resource);
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
    SqlSession sqlSession = sqlSessionFactory.openSession();
    try {
        //获取接口的实现类对象(在 EmployeeMapper 中 nameSpace 映射接口全类名，在 select.id 中映射方法)
        //会为接口自动创建代理对象，代理对象进行增删改查操作
        EmployeeMapper mapper = sqlSession.getMapper(EmployeeMapper.class);
        Employee empById = mapper.getEmpById(1);
        System.out.println(empById);
    } finally {
        sqlSession.close();
    }
}
```

### 1.3 总结

> 1. 接口式编程
>
> - 原生：Dao -> DaoImple
>
> - mybastis：Mapper -> xxMapper.xml
>
> 1. sqlSession 代表和数据库的一次会话，用完需关闭
>
> 3. sqlSession 和 connection 都是**非线程安全**，每次使用都要获取新的对象
> 4. mapper 没有实现类，但 mybatis 会为这个接口生成一个代理对象（将接口与 xml 文件绑定）：`EmployeeMapper mapper = sqlSession.getMapper(EmployeeMapper.class);`
>
> 5. 两个重要的配置文件：
>
> - ==mybatis 全局配置文件==：包含数据库连接信息，事务管理信息，系统运行环境信息等
> - ==sql 映射文件==：保存了每一个 sql 语句的映射信息

---

## 2. 全局配置文件

### 2.1 properties 标签（obsolete）

```xml
mybatis-config.xml
=========================================================================
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!--
    mybatis 可以是有 properties 来引入外部 properties 配置文件的内容
    	- resource：引入类路径下资源
    	- url：引入网络路径或磁盘路径下的资源
    -->
    <properties resource="dbconfig.properties"/>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <!-- 通过 ${} 从 properties 文件中获取值 -->
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>
    
    <mappers>
        <mapper resource="EmployeeMapper.xml"/>
    </mappers>
</configuration>
```

```properties
dbconfig.properties
===========================================================================
jdbc.driver=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/mybatis?allowPublicKeyRetrieval=true
jdbc.username=root
jdbc.password=root
```

### 2.2 setting 标签

```xml
<settings>
    <!-- 开启下划线命名（DB中）转驼峰命名（实例 Bean 中）-->
    <setting name="mapUnderscoreToCamelCase" value="true"/>
</settings>
```

### 2.3 typeAliases

```xml
 <!-- 可以为 java 类型起别名（别名 不区分大小写） -->
    <typeAliases>
        <!--
            typeAlias：为某个 java 类型起名
            type：指定要起别名的类型全类名，默认别名为类型小写——employee
            alias：自定义名字
        -->
        <typeAlias type="bean.Employee" alias="customizeName"/>

        <!-- 为某个包下所有类批量起别名 -->
        <package name="bean"/>

        <!-- 可以在 实例 bean 中使用 @Alias 注解为某个类型具体指定别名 -->

    </typeAliases>
```

### 2.4 typeHandlers

### 2.5 plugins

> Executor，ParameterHandler，ResultSetHandler，StatumentHandler

### 2.6 environments

```xml
<!--environments：可以配置多种环境，default 指定使用某种环境-->
<environments default="development">
<!--
    environment：配置一个具体的环境信息，id 代表当前环境的唯一标识
        必须有：
            1. transactionManager：事务管理器
                - type：事务管理器类型，JDBC（JdbcTransactionFactory） / MANAGED（ManagedTransactionFactory）
                （Configuration.class 中注册了别名）
                - 自定义事务管理器：实现 TransactionManager 接口，type 指定为全类名
            2. dataSource：数据源
                - type：数据源类型，UNPOOLED，POOLED，JNDI
                - 自定义数据源：实现 DataSourceFactory 接口，type 指定为全类名
-->
    <environment id="development">
        <transactionManager type="JDBC"/>
        <dataSource type="POOLED">
            <property name="driver" value="${jdbc.driver}"/>
            <property name="url" value="${jdbc.url}"/>
            <property name="username" value="${jdbc.username}"/>
            <property name="password" value="${jdbc.password}"/>
        </dataSource>
    </environment>

    <environment id="test">
        <transactionManager type=""/>
        <dataSource type=""/>
    </environment>
</environments>
```

### 2.7 databaseIdProvider

> 1. 在全局配置文件中添加 databaseIdProvider 标签
>
>    ```xml
>    <!--
>        支持多数据库厂商
>        type="DB_VENDOR"：VendorDatabaseIdProvider
>            - 得到数据库厂商的标识（驱动 getDatabaseProductName()）
>    -->
>    <databaseIdProvider type="DB_VENDOR">
>        <!-- 为不同数据库厂商起别名 -->
>        <property name="MySQL" value="mysql"/>
>        <property name="Oracle" value="oracle"/>
>        <property name="SQL Server" value="sqlserver"/>
>    </databaseIdProvider>
>    ```
>
> 2. 在 mapper.xml 文件中声明使用的 databaseId
>
>    ```xml
>    <mapper namespace="dao.EmployeeMapper">
>        <select id="getEmpById" resultType="bean.Employee">
>            select id, last_name, email, gender from tbl_employee where id = #{id}
>        </select>
>        <!-- 如果有不带标识的和带标识的，查询 mysql 时会使用更精确的语句，即会覆盖上面的语句 -->
>        <select id="getEmpById" resultType="bean.Employee" databaseId="mysql">
>            select id, last_name, email, gender from tbl_employee where id = #{id}
>        </select>
>        <select id="getEmpById" resultType="bean.Employee" databaseId="oracle">
>            select id, last_name, email, gender from tbl_employee where id = #{id}
>        </select>
>    </mapper>
>    ```
>
> 3. 修改 environments 的 default 属性，即会自动切换 dataBase

### 2.8 ==mappers==

```xml
<!-- 将 sql 映射注册到全局配置中 -->
<mappers>
    <!--
    mapper：注册一个 sqp 映射
    	注册配置文件：
    		- resource：引用路径下的 sql 映射文件（mybatis/mapper/testMapper.xml）
    		- url：引用网络路径或磁盘路径下的 sql 映射文件（file:///var/mappers/testMapper.xml）
		注册接口：
    		- class：直接引用接口
    			1. sql 映射文件名必须和接口同名，并放在同一目录下
    			2. 没有 sql 映射文件，所有 sql 都是利用注解写在接口上
    			推荐写 sql 映射文件，简单的 Dao 接口可以使用注解
    -->
    <!--<mapper resource="EmployeeMapper.xml"/>-->
    <mapper class="dao.EmployeeMapper"></mapper>
    <!--
        批量注册
		- 相当于类名注册
        - 需要在资源文件夹下建一个同名包，并把 mapper.xml 文件放入
    -->
    <package name="dao"/>
</mappers>
```

### 总结

> 标签有先后顺序：
>
> - properties, settings, typeAliases, typeHandlers, objectFactory, objectWrapperFactory, reflectorFactory, plugins, environments, databaseIdProvider, mappers

---

## 3. Mybatis 映射文件

### 3.1 增删改查

> 1. mapper 接口
>
>    ```java
>    public interface EmployeeMapper {
>        
>        public Employee getEmpById(Integer id);
>        public Boolean addEmp(Employee employee);
>        public void updateEmp(Employee employee);
>        public Boolean deleteEmpById(Integer id);
>        
>    }
>    ```
>
> 2. mapper 映射文件
>
>    ```xml
>    <mapper namespace="com.mybatis.dao.EmployeeMapper">
>        <select id="getEmpById" resultType="com.mybatis.bean.Employee">
>            select id, last_name, email, gender from tbl_employee where id = #{id}
>        </select>
>    
>        <!-- parameterType：可以省略 -->
>        <insert id="addEmp" parameterType="com.mybatis.bean.Employee">
>            insert into tbl_employee(last_name,email,gender)
>            values(#{lastName},#{email},#{gender})
>        </insert>
>    
>        <update id="updateEmp">
>            update tbl_employee
>            set last_name=#{lastName},email=#{email},gender=#{gender}
>            where id=#{id}
>        </update>
>    
>        <delete id="deleteEmpById">
>            delete from tbl_employee where id=#{id}
>        </delete>
>    </mapper>
>    ```
>
> 3. 执行方法
>
>    ```java
>    @Test
>    public void test03() throws IOException {
>        String resource = "mybatis-config.xml";
>        InputStream inputStream = Resources.getResourceAsStream(resource);
>        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
>        // 获取到的 sqlSession 不会自动提交数据，需要手动提交
>        // sqlSessionFactory.openSession(true) 获取到的 sqlSession会自动提交，不需手动提交数据
>        SqlSession sqlSession = sqlSessionFactory.openSession();
>        // 增删改查
>        // mybatis 允许增删改查直接定义以下类型返回值：Integer，Long，Boolean
>        // 只需在 mapper 接口的方法上修改返回值即可
>        try{
>            EmployeeMapper mapper = sqlSession.getMapper(EmployeeMapper.class);
>            // 添加
>            // mapper.addEmp(new Employee(null, "tom", "a@test/com", "1"));
>            // 修改
>            // mapper.updateEmp(new Employee(1, "tom", "a@test/com", "1"));
>            // 删除
>            System.out.println(mapper.deleteEmpById(1));
>            // 手动提交
>            sqlSession.commit();
>        } finally {
>            sqlSession.close();
>        }
>    }
>    ```

### 3.2 insert 获取自增主键的值

> **1. mySQL 获取主键**
>
> mybatis 支持自增主键，自增主键的获取也是利用 statement.getGeneratedKeys()
>
> - **useGeneratedKeys**：开启获取自增主键值策略
> - **keyProperty**：指定对应的主键属性，获取到主键值后，将这个值封装给 javaBean 的哪个属性
>
> ```xml
> <insert id="addEmp" parameterType="com.mybatis.bean.Employee" useGeneratedKeys="true" keyProperty="id">
>     insert into tbl_employee(last_name,email,gender)
>     values(#{lastName},#{email},#{gender})
> </insert>
> ```
>
> ```java
> Employee employee = new Employee(null, "tom", "a@test/com", "1");
> mapper.addEmp(employee);
> System.out.println(employee.getId());
> ```
>
> **2. Oracle 获取主键**
>
> Oracle 不支持自增，使用序列来模拟自增，每次插入的数据的主键是从序列中拿到的值
>
> ```xml
> <insert id="addEmp" parameterType="com.mybatis.bean.Employee">
>     <!--
>         keyProperty：查出的主键值封装给 javaBean 的哪个属性
>         order：指定当前 sql 执行的顺序
>         resultType：指定数据返回值类型
>         Before 运行顺序：
>             1. 运行 selectKey 查询下一个 id 的 sql， 查出后封装给 javeBean 的id属性
>             2. 再运行插入的 sql，就可以取出 id 属性对应的值
>     -->
>     <selectKey keyProperty="id" order="BEFORE" resultType="Integer">
>         select tbl_employee_seq.nextval from tbl
>     </selectKey>
>     insert into tbl_employee(id,last_name,email,gender)
>     values(#{id},#{lastName},#{email},#{gender})
> </insert>
> 
> <insert id="addEmp" parameterType="com.mybatis.bean.Employee">
>     <!--
>         After 运行顺序：
>             1. 先运行插入的 sql，从序列中取出 id 值
>             2. 再运行 selectKey 查询当前 id 的 sql， 查出后封装给 javeBean 的id属性
>     -->
>     <selectKey keyProperty="id" order="AFTER" resultType="Integer">
>         select tbl_employee_seq.currval from tbl
>     </selectKey>
>     insert into tbl_employee(id,last_name,email,gender)
>     values(tbl_employee_seq.nextval,#{lastName},#{email},#{gender})
> </insert>
> ```

### 3.3 参数处理

> **1. 单个参数**（mybatis 不会做特殊处理）
>
> - #{参数名}：取出参数值，参数名可任意设置
>
> - `select id, last_name, email, gender from tbl_employee where id = #{id}`
>
> **2. 多个参数**（mybatis 会做特殊处理）
>
> 1. 多个参数会自动封装成一个 map，取值时按 key 值获取
>
>    - key：param1...paramN，或者arg0...argN
>    - value：传入的参数值
>
>    ```xml
>    <select id="getEmpByIdAndGender" resultType="com.mybatis.bean.Employee">
>        select id, last_name, email, gender from tbl_employee where id = #{arg0} and gender=#{arg1}
>    </select>
>    ```
>
>    ```java
>    public Employee getEmpByIdAndGender(Integer id, String gender);
>    ```
>
> 2. ==命名参数==：使用 `@Param` 注解指定 key 值时，即使只有一个参数，也会封装到 map 中
>
>    ```xml
>    <select id="getEmpByIdAndGender" resultType="com.mybatis.bean.Employee">
>        select id, last_name, email, gender from tbl_employee where id = #{id} and gender=#{gender}
>    </select>
>    ```
>
>    ```java
>    public Employee getEmpByIdAndGender(@Param("id")Integer id, @Param("gender")String gender); 
>    ```
>
> 3. POJO
>
>    1. 如果多个参数正好是业务逻辑的数据模型，可以直接传入 POJO
>
>       - #{属性名}：取出传入 POJO 的属性值
>
>    2. 如果多个参数不是业务模型中的数据，也可以传入 map（不推荐）
>
>       - #{key}：取出传入 map 的属性值
>
>       ```xml
>       <select id="getEmpByMap" resultType="com.mybatis.bean.Employee">
>           select id, last_name, email, gender from tbl_employee where id = #{id} and gender=#{gender}
>       </select>
>       ```
>
>       ```java
>       public Employee getEmpByMap(Map<String, Object> map);
>       ```
>
>       ```java
>       EmployeeMapper mapper = sqlSession.getMapper(EmployeeMapper.class);
>       Map<String, Object> map = new HashMap<>();
>       map.put("id", 3);
>       map.put("gender", "1");
>       Employee empById = mapper.getEmpByMap(map);
>       ```
>
>    3. 如果多个参数不是业务模型中的数据，但是经常要用，推荐编写一个 TO（Transfer Object）数据传输对象

#### 3.3.1 几个知识点

> 1. `public Employee getEmp(@Param("id")Integer id, @Param("gender")String gender)`
>    - ==xml 取值==：id -> #{id / param1}；lastName -> #{param2}
> 2. `public Employee getEmp(Integer id, @Param("e")Employee emp)`
>    - ==xml 取值==：id -> #{param1}；lastName -> #{param2.lastName / e.lastName}
> 3. `public Employee getEmp(List<Integer> ids)`
>    - 如果是 Collection（List，Set 等）类型，或者是数组，也会特殊处理，把传入的集合或数组封装在 map 中
>      - key：
>        - Collection -> collection
>        - List -> collection / list
>        - 数组 -> array
>    - ==xml 取值==：取出第一个 id 的值：**#{list[0]}**
> 4. `public Employee getEmpByList(Integer id, List<Object> list)`
>    - ==xml 取值==：取出 list 中第一个值：**#{param2[0]}**

#### 3.3.2 源码：参数封装 map 过程

> MapperProxy.class -> invoke()
>
> MapperMethod.class -> execute()
>
> ```java
> ParamNameResolver.class
>     
> // 解析 mapper 接口方法的参数名
> public ParamNameResolver(Configuration config, Method method) {
>    ... ...
> 
>             for(int var11 = 0; var11 < var10; ++var11) {
>                 Annotation annotation = var9[var11];
>                 // 获取标记了 @Param 注解的参数值
>                 if (annotation instanceof Param) {
>                     this.hasParamAnnotation = true;
>                     name = ((Param)annotation).value();
>                     break;
>                 }
>             }
> 
>             if (name == null) {
>                 // <setting name="useActualParamName" value=""/> 默认为 true
>                 if (this.useActualParamName) {
>                     // 获得 arg0...argN
>                     name = this.getActualParamName(method, paramIndex);
>                 }
> 				// 不开启 ActualParamName 时，则参数为 0...N
>                 if (name == null) {
>                     name = String.valueOf(map.size());
>                 }
>             }
> 			// 每次解析一个参数，就保存参数索引和 name 的值
>     			// 标注了 @Param 时即为 @Param 的 value
>     			// 没有标注时
>             map.put(paramIndex, name);
>         }
>     }
> 	// {0=arg0, 1=arg1}
>     this.names = Collections.unmodifiableSortedMap(map);
> }
>     
> public Object getNamedParams(Object[] args) {
>     // name 为 EmployeeMapper 的接口类中
>     int paramCount = this.names.size();
>     if (args != null && paramCount != 0) {
>         // 如果只有一个元素，并且没有 @Param 注解
>         if (!this.hasParamAnnotation && paramCount == 1) {
>             // 拿到 arg[0]
>             Object value = args[(Integer)this.names.firstKey()];
>             return wrapToMapIfCollection(value, this.useActualParamName ? (String)this.names.get(0) : null);
>         // 有多个元素或有 @Param 标注时
>         } else {
>             Map<String, Object> param = new ParamMap();
>             int i = 0;
> 			// names map：{arg0=id, arg1=gender}
>             for(Iterator var5 = this.names.entrySet().iterator(); var5.hasNext(); ++i) {
>                 Entry<Integer, String> entry = (Entry)var5.next();
>                 // param map：{arg0=args[0], arg1=args[1]}
>                 param.put(entry.getValue(), args[(Integer)entry.getKey()]);
>                 // 并且将每一个参数使用 param1...paramN 作为 key 也保存到 param map中
>                 String genericParamName = "param" + (i + 1);
>                 if (!this.names.containsValue(genericParamName)) {
>                     param.put(genericParamName, args[(Integer)entry.getKey()]);
>                 }
>             }
> 
>             return param;
>         }
>     } else {
>         return null;
>     }
> }
> ```
>
> ==总结==
>
> 1. 若标注了 @Param 注解，则可以使用 #{属性名} / #{param1...paramN}
> 2. 若没有标注 @Param 注解，则可以使用 #{arg0...arg1} / #{param1...paramN}
> 3. 若设置了 `<setting name="useActualParamName" value="false"/>`，则可以使用 #{arg0...arg1} / #{0...N}

#### 3.3.3 参数值的获取

> 1. **#{}**：可以获取 map 中的值或者 POJO 对象属性的值
>
> 2. **${}**：可以获取 map 中的值或者 POJO 对象属性的值
>
> ==区别==
>
> - #{}：是以预编译的形式（? 占位符），将参数设置到 sql 语句中，利用 PreparedStatument，可防止 sql 注入
> - ${}：取出的值直接拼装在 sql 语句中，会有安全问题
> - 大多情况下都应该使用 #{}，当想动态替换 sql 语句中的字段（参数以外）时，可以使用 ${}
>   - 按照年份分表：`select * from ${year}_salary where xxx`
>   - 排序：`select * from table order by ${name} ${order}`
>
> ==#{}：更丰富的用法==
>
> - 规定一些规则：javaType，jabcType，mode（存储过程），numericScale，resultMap，typeHandler，jdbcTypeName，expression
>
>   - **jdbcType**：通常需要在某种特定条件下设置
>
>     - 在数据为 null 时，有些数据库不能识别 mybatis 对 null 的默认处理
>
>       - 比如 oracle，会抛出 JdbcType OTHER：无效的类型，因为 mybatis 对所有的 null 都映射的是原生 Jdbc 的 OTHER 类型，oracle 不能识别，mySQL 可以正常识别
>
>       - 解决：
>
>         1. ```sql
>            select id, last_name, email, gender from tbl_employee where id = #{id} and gender=#{gender, jdbcType=OTHER}
>            ```
>
>         2. 全局变量的 setting 中设置：jdbcTypeForNull=NULL
>
>         

### 3.4 select

> **id**：唯一标识符
>
> - 需要和接口的方法名一致
>
> **parameterType**：参数类型
>
> - 可以不传，mybatis 会根据 TypeHandler 自动推断
>
> **resultType**：返回值类型
>
> - 别名或者全类名

#### 3.4.1 返回值为 List

> 如果返回的是集合，则 resultType 需要指定为==集合中元素的类型==。不能和 resultMap 同时使用
>
> ```xml
> <!-- 返回的是集合时，resultType 中指定集合中元素的类型 -->
> <select id="getEmp" resultType="com.mybatis.bean.Employee">
>     select * from tbl_employee where gender=#{gender}
> </select>
> ```
>
> ```java
> public List<Employee> getEmp(String gender);
> ```

#### 3.4.2 封装 map

> - 返回一条记录的 map：key 是列名，值是对应的值
>
> - 使用 `resultType="Map"`
>
>   ```xml
>   <select id="getEmpMap" resultType="Map">
>       select * from tbl_employee where id = #{id}
>   </select>
>   ```
>
>   ```java
>   public Map<String, Object> getEmpMap(Integer id);
>   ```
>
> - 多条记录封装一个 map：key 是记录的主键，值是封装后的 javaBean
>
> - 使用 `@MapKey` 注解
>
>   ```xml
>   <select id="getEmpManyMap" resultType="Map">
>       select * from tbl_employee where gender=#{gender}
>   </select>
>   ```
>
>   ```java
>   // 告诉 mybatis 封装这个 map 时用哪个属性作为 key
>   @MapKey("id")
>   public Map<Integer, Employee> getEmpManyMap(String gender);
>   ```

#### 3.4.3 resultMap

##### 1 自定义结果映射规则

> 1. 全局 setting 设置
>
>    - `autoMappingBehavior` 默认是 PARTIAL，开启自动映射功能，唯一的要求是列名和 javaBean 属性名一致
>    - 如果 `autoMappingBehavior` 设置为 null 则会取消自动映射
>    - 数据库字段命名规范，POJO 属性符合驼峰命名法，如 ACOLUMN -> aColumn，可以设置 `mapUnderScoreToCamelCase=true` 开启自动驼峰命名规则映射功能
>
> 2. 在 sql 语法中声明别名
>
>    ```sql
>    select id, last_name lastName, email, gender from tbl_employee where id = #{id}
>    ```
>
> 3. 自定义 resultMap，实现高级结果映射
>
>    ```xml
>    <!--
>        自定义某个 javaBean 的封装规则
>        type：自定义规则的 java 类型
>        id：唯一 id
>    -->
>    <resultMap id="myEmp" type="com.mybatis.bean.Employee">
>        <!--
>            指定主键列的封装规则
>                用 id 定义主键底层会有优化
>                column：指定哪一列
>                property：指定对应的 javaBean 属性
>        -->
>        <id column="id" property="id"></id>
>        <!-- 定义普通列的封装规则 -->
>        <result column="last_name" property="lastName"></result>
>        <!-- 其他不指定的列会自动封装，推荐写 resultMap 的话就把全部的映射规则都写上 -->
>    </resultMap>
>                                                 
>    <!-- resultMap：自定义结果集映射规则 -->
>    <select id="getEmpById" resultMap="myEmp">
>        select * from tbl_employee where id = #{id}
>    </select>
>    ```

##### 2 关联查询

> 1. 级联属性指定关联查询的结果
>
>    ```xml
>    <resultMap id="myEmp" type="com.mybatis.bean.Employee">
>        <id column="id" property="id"></id>
>        <result column="last_name" property="lastName"></result>
>        <result column="gender" property="gender"></result>
>        <result column="email" property="email"></result>
>        <!-- 联合查询：级联属性封装结果集 -->
>        <result column="d_id" property="dept.id"></result>
>        <result column="dept" property="dept.departmentName"></result>
>    </resultMap>
>    
>    <select id="getEmpAndDep" resultMap="myEmp">
>        SELECT
>        e.id id, e.last_name last_name, e.gender gender, e.email email, d.id d_id, d.dept_name dept
>        FROM tbl_employee e LEFT JOIN tbl_dept d ON e.id=d.id WHERE e.id=1
>    </select>
>    ```
>
> 2. **association**
>
>    1. 定义关联的单个对象的封装规则
>
>       ```xml
>       <resultMap id="myEmp" type="com.mybatis.bean.Employee">
>           <id column="id" property="id"></id>
>           <result column="last_name" property="lastName"></result>
>           <result column="gender" property="gender"></result>
>           <result column="email" property="email"></result>
>           <!--
>               association 可以指定联合的 javaBean 对象
>                   - property：指定哪个属性是联合的对象
>                   - javaType：指定这个属性对象的类型
>           -->
>           <association property="dept" javaType="com.mybatis.bean.Department">
>               <id column="d_id" property="id"></id>
>               <result column="dept" property="departmentName"></result>
>           </association>
>       </resultMap>
>       
>       <select id="getEmpAndDep" resultMap="myEmp">
>           SELECT
>           e.id id, e.last_name last_name, e.gender gender, e.email email, d.id d_id, d.dept_name dept
>           FROM tbl_employee e LEFT JOIN tbl_dept d ON e.id=d.id WHERE e.id=1
>       </select>
>       ```
>
>    2. 分步查询
>
>       ```xml
>       <mapper namespace="com.mybatis.dao.EmployeeMapper">
>       
>           <!--
>           1. 按照员工 id 查询员工信息
>           2. 根据查询员工信息中的 d_id 值去部门表查出部门信息
>           3. 将部门设置到员工中
>           -->
>           <resultMap id="myEmpByStep" type="com.mybatis.bean.Employee">
>               <id column="id" property="id"></id>
>               <result column="last_name" property="lastName"></result>
>               <result column="email" property="email"></result>
>               <result column="gender" property="gender"></result>
>               <!--
>               association 定义关联对象的封装规则
>                   property：指定 javaBean 中待封装的属性
>                   select：表明当前属性是调用 select 指定的方法查出的结果
>                   column：指定将哪一列的值传给这个方法
>               流程：
>       			先查出员工的信息
>                   再使用 select 指定的方法（传入 column 指定的这列参数的值）查出对象，并封装给 property 指定属性
>               -->
>               <association property="dept"
>                            select="com.mybatis.dao.DepartmentMapper.getDeptById"
>                            column="d_id"
>               ></association>
>           </resultMap>
>           <select id="getEmpByIdStep" resultMap="myEmpByStep">
>               select * from tbl_employee where id=#{id}
>           </select>
>       </mapper>
>       ```
>
>       ```xml
>       <mapper namespace="com.mybatis.dao.DepartmentMapper">
>           <select id="getDeptById" resultType="com.mybatis.bean.Department">
>               select id, dept_name deartmentName from tbl_dept where id=#{id}
>           </select>
>       </mapper>
>       ```
>
>    3. 延迟加载
>
>       例：只有当使用到 Employee 的 Department 属性时，才会发送查询 department 信息的 sql 语句
>
>       ```xml
>       <!-- 显示的指定每个需要更改的配置的值，防止版本更新带来的问题 -->
>       <setting name="lazyLoadingEnabled" value="true"/>
>       <setting name="aggressiveLazyLoading" value="false"/>
>       ```
>
> 3. **collection**
>
>    1. 定义关联集合类型的属性的封装规则
>
>       例：查询部门时将部门对应的所有员工信息也查询出来
>
>       ```xml
>       <resultMap id="myDept" type="com.mybatis.bean.Department">
>           <id column="did" property="id"></id>
>           <result column="dept_name" property="departmentName"></result>
>           <!--
>               collection 定义关联集合类型的属性的封装规则
>                   - ofType：指定集合里面元素的类型
>               -->
>           <collection property="emps" ofType="com.mybatis.bean.Employee">
>               <!-- 定义这个集合中元素的封装规则 -->
>               <id column="eid" property="id"></id>
>               <result column="last_name" property="lastName"></result>
>               <result column="email" property="email"></result>
>               <result column="gender" property="gender"></result>
>           </collection>
>       </resultMap>
>       <select id="getDeptByIdPlus" resultMap="myDept">
>           SELECT
>           d.id did, d.dept_name dept_name, e.id eid, e.last_name last_name, email email, e.gender gender
>           FROM tbl_dept d
>           LEFT JOIN tbl_employee e
>           ON d.id=e.d_id
>           WHERE d.id=#{id}
>       </select>
>       ```
>
>       ```java
>       public class Department {
>           private Integer id;
>           private String departmentName;
>           private List<Employee> emps;
>       }
>       ```
>
>    2. 分布查询 & 延迟查询
>
>       ```xml
>       <mapper namespace="com.mybatis.dao.DepartmentMapper">
>           <resultMap id="myDeptStep" type="com.mybatis.bean.Department">
>               <id column="id" property="id"></id>
>               <result column="dept_name" property="departmentName"></result>
>               <collection property="emps" select="com.mybatis.dao.EmployeeMapper.getEmpsByDeptId" column="id"></collection>
>           </resultMap>
>           <select id="getDeptByIdStep" resultMap="myDeptStep">
>               select id, dept_name from tbl_dept where id=#{id}
>           </select>
>       </mapper>
>       ```
>
>       ```xml
>       <mapper namespace="com.mybatis.dao.EmployeeMapper">
>           <select id="getEmpsByDeptId" resultType="com.mybatis.bean.Employee">
>               select * from tbl_employee where d_id=#{deptId}
>           </select>
>       </mapper>
>       ```
>
> 4. 多列的值传给 association 和 collection 的 colum 的属性
>
>    - `column="key1=column1, key2=column2"`
>
>    ```xml
>    <collection property="emps" select="com.mybatis.dao.EmployeeMapper.getEmpsByDeptId" column="{deptId=id}"></collection>
>    ```
>
> 5. association 和 collection 的`fetchType` 属性
>
>    1. lazy（默认）：表示开启延迟加载
>    2. eager：即使全局设置开启了延迟加载，设置为 eager 时也会变成立即加载

##### 3 鉴别器

> - mybatis 可以使用 discriminator 判断某列的值，然后根据该值改变封装行为
>
> - 例：封装 Employee
>
>   - 如果是女生，就把部门信息查出来
>   - 如果是男生，把 last_name 的值赋给 email
>
>   ```xml
>   <resultMap id="MyEmpDis" type="com.mybatis.bean.Employee">
>       <id column="id" property="id"></id>
>       <result column="last_name" property="lastName"></result>
>       <result column="email" property="email"></result>
>       <result column="gender" property="gender"></result>
>                                 
>       <!--
>           - column：指定判定的列名
>           - javaType：列值对应的 java 类型
>           -->
>       <discriminator javaType="string" column="gender">
>           <!-- resultType 不能缺少，or resultMap -->
>           <case value="0" resultType="com.mybatis.bean.Employee">
>               <association property="dept"
>                            select="com.mybatis.dao.DepartmentMapper.getDeptByIdPlus"
>                            column="d_id"
>                            />
>           </case>
>                                 
>           <case value="1" resultType="com.mybatis.bean.Employee">
>               <result column="last_name" property="email"></result>
>           </case>
>       </discriminator>
>                                 
>   </resultMap>
>   <select id="getEmpDis" resultMap="MyEmpDis">
>       select * from tbl_employee where id=#{id}
>   </select>
>   ```

---

## 4. 动态 SQL

### 4.1 If

```xml
<!-- 查询员工，要求携带了哪个字段查询条件就带上这个字段的值 -->
<select id="getEmpsByConditionIf" resultType="com.mybatis.bean.Employee">
    select * from tbl_employee
    where
    <!--
        test：判断表达式（OGNL）
            - 从参数中取值进行判断
            - 遇见特殊符号使用转义字符
        -->
    <if test="id!=null">
        id=#{id}
    </if>
    <if test="lastName!=null and lastName!=''">
        and last_name=#{lastName}
    </if>
    <if test="email!=null and email.trim()!=''">
        and email=#{email}
    </if>
    <if test="gender==0 or gender==1">
        and gender=#{gender}
    </if>
</select>
```

### 4.2 where

> 再上一个例子中查询时，如果 id 为空，则 sql 拼装会有问题
>
> 解决：
>
> 1. 给 where 后面加上 1=1，之后的条件都 and XXX
>
> 2. mybatis 使用 where 标签来将所有查询条件包括在内。mybatis 就会将 where 中拼装的多出来的 and 和 or 去掉
>
>    - ==注意==：只会去掉第一个多出来的 and 和 or，and写在末尾会出问题
>
>    ```xml
>    <select id="getEmpsByConditionWhere" resultType="com.mybatis.bean.Employee">
>        select * from tbl_employee
>        <where>
>            <if test="id!=null">
>                id=#{id}
>            </if>
>            <if test="lastName!=null and lastName!=''">
>                and last_name=#{lastName}
>            </if>
>            <if test="email!=null and email.trim()!=''">
>                and email=#{email}
>            </if>
>            <if test="gender==0 or gender==1">
>                and gender=#{gender}
>            </if>
>        </where>
>    </select>
>    ```

### 4.3 trim

> - 自定义 sql 拼装规则
>
> - 当 gender 为空时，会自动去掉 email 后的 and
>
>   ```xml
>   <select id="getEmpsByConditionTrim" resultType="com.mybatis.bean.Employee">
>       select * from tbl_employee
>       <!--
>           trim 标签体中是整个字符串拼串后的结果
>           - prefix：给拼串后的字符串加一个前缀
>           - prefixOverrides：前缀覆盖，去掉整个字符串前面多余的字符
>           - suffix：给拼串后的字符串加一个后缀
>           - suffixOverrides：后缀覆盖，去掉整个字符串后面多余的字符
>           -->
>       <trim prefix="where" prefixOverrides="" suffix="" suffixOverrides="and">
>           <if test="id!=null">
>               id=#{id} and
>           </if>
>           <if test="lastName!=null and lastName!=''">
>               last_name=#{lastName} and
>           </if>
>           <if test="email!=null and email.trim()!=''">
>               email=#{email} and
>           </if>
>           <if test="gender==0 or gender==1">
>               gender=#{gender}
>           </if>
>       </trim>
>   </select>
>   ```

### 4.4 choose

> - 分支选择（相当于 switch - case）
>
>   ```xml
>   <select id="getEmpsByConditionChose" resultType="com.mybatis.bean.Employee">
>       select * from tbl_employee
>       <where>
>           <choose>
>               <when test="id!=null">
>                   id=#{id}
>               </when>
>               <when test="lastName!=null">
>                   last_name=#{lastName}
>               </when>
>               <when test="email">
>                   email=#{email}
>               </when>
>               <otherwise>
>                   gender=0
>               </otherwise>
>           </choose>
>       </where>
>   </select>
>   ```

### 4.5 set

> - 封装修改条件
>
> - 例：当对应属性不为空就更新该属性，set 会自动删除多余的 ","
>
>   ```xml
>   <update id="updateEmp">
>       update tbl_employee
>       <set>
>           <if test="lastName!=null">
>               last_name=#{lastName},
>           </if>
>           <if test="email!=null">
>               email=#{email},
>           </if>
>           <if test="gender!=null">
>               gender=#{gender},
>           </if>
>       </set>
>       <where>
>           id=#{id}
>       </where>
>   </update>
>   ```
>
> - 或用 trim 标签删除多余的符号
>
>   ```xml
>   <update id="updateEmp">
>       update tbl_employee
>       <trim prefix="set" suffixOverrides=",">
>           <if test="lastName!=null">
>               last_name=#{lastName},
>           </if>
>           <if test="email!=null">
>               email=#{email},
>           </if>
>           <if test="gender!=null">
>               gender=#{gender},
>           </if>
>       </trim>
>       <where>
>           id=#{id}
>       </where>
>   </update>
>   ```

### 4.6 foreach

> - 用来解决 sql 语句中 in(val1, val2) 的字段
>
>   ```xml
>   <select id="getEmpsByConditionForeach" resultType="com.mybatis.bean.Employee">
>       select * from tbl_employee where id in
>       <!--
>           collection：指定要遍历的集合
>               - list 类型的参数会特殊处理封装在 map 中，map 的 key 就叫 list
>           item：将当前遍历出的元素赋值给指定的变量
>           separator：元素之间的分隔符
>           open：遍历出所有结果拼接一个开始的字符
>           close：遍历出所有结果拼接一个结束的字符
>           index：
>               - 遍历 list 时是索引，item 是当前值
>               - 遍历 map 时表示 map 的key，item 是值
>           #{变量名} 就能取出当前变量的值
>           -->
>       <foreach collection="ids" item="item" separator="," open="(" close=")">
>           #{item}
>       </foreach>
>   </select>
>   ```
>
>   ```java
>    List<Employee> getEmpsByConditionForeach(@Param("ids") List<Integer> list);
>   ```
>
> - 批量保存
>
>   - mySQL
>   
>     ```xml
>     <insert id="addEmps">
>         <!-- mySql下批量保存，可以 foreach 遍历，mySql支持 values(),() 语法 -->
>         insert into tbl_employee(last_name,email,gender,d_id) values
>         <foreach collection="emps" item="emp" separator=",">
>             (#{emp.lastName},#{emp.email},#{emp.gender},#{emp.dept.id})
>         </foreach>
>     </insert>
>     <!-- url 中开启 allowMultiQeuries=true，支持分号连接多个 insert 语句 -->
>     <!-- 这种分隔多个 sql 可以用于其他批量操作（删除，修改） -->
>     <insert id="addEmps">
>         <foreach collection="emps" item="emp" separator=";">
>             insert into tbl_employee(last_name,email,gender,d_id) values
>             (#{emp.lastName},#{emp.email},#{emp.gender},#{emp.dept.id})
>         </foreach>
>     </insert>
>     ```
>   
>     ```java
>     public void addEmps(@Param("emps")List<Employee> emps);
>     ```
>   
>   - Oracle
>   
>     - 支持的插入方式
>   
>       ```mysql
>       begin
>       	insert into tbl_employee(employee_id,last_name,email)
>       	values(employee_seq.nextval,'test_001','test_001@test.com');
>       	insert into tob_employee(employee_id,last_name,email)
>       	values(employee_seq.nextval,'test_002','test_002@test.com');
>       end;
>       ```
>   
>       ```mysql
>       insert into tbl_employee(employee_id,last_name,email)
>       	select employee_seq.nextval,last_name,email from(
>           	select 'test_001' last_name,'test_001@test.com' email from dual
>               union
>               select 'test_002' last_name,'test_002@test.com' email from dual
>           )
>       ```
>   
>     - foreach 实现
>   
>       ```xml
>       <insert id="addEmps" databaseId="oracle">
>           <foreach collection="emps" item="emp" open="begin" close="end;">
>               insert into tbl_employee(employee_id,last_name,email) values
>               (employee_seq.nextval,#{emp.lastName},#{emp.email});
>           </foreach>
>       </insert>
>       ```
>   
>       ```xml
>       <insert id="addEmps" databaseId="oracle">
>           insert into tbl_employee(employee_id,last_name,email)
>           	select employee_seq.nextval,last_name,email from
>           <foreach collection="emps" item="emp" open="(" close=")" separator="union">
>               select #{emp.lastName} last_name,#{emp.email} email from dual
>           </foreach>
>       </insert>
>       ```

### 4.7 内置参数

> 1. **_parameter**：代表整个参数
>
>    - 单个参数：_parameter 就是这个参数
>
>      ```xml
>      select * from tbl_employee
>      <if test="_parameter!=null">
>          where last_name=#{_parameter.lastName}
>      </if>
>      ```
>
>    - 多个参数：参数被封装为一个 map，_parameter 就是这个 map
>
>      - 注意：当使用 @Param 注解时，一定会生成两个参数，一个 key 为 @Param 的值，另一个 key 为 param1
>
> 2. **_databaseId**
>
>    - 如果配置了 databaseIdProvider 标签，则 _databaseId 代表当前数据库的别名，例：oracle
>
>      ```xml
>      <select id="getEmpsTestInnerParameter" resultType="com.mybatis.bean.Employee">
>          <if test="_databaseId=='mysql'">
>              select * from tbl_employee
>          </if>
>          <if test="_databaseId=='oracle'">
>              select * from employees
>          </if>
>      </select>
>      ```

### 4.8 绑定——bind（推荐直接在传参数时传入'%模糊字符%'）

> bind：可以将 OGNL 表达式的值绑定到一个变量中，方便后来引用这个变量的值
>
> ```xml
> <select id="getEmpsTestBind" resultType="com.mybatis.bean.Employee">
>     <bind name="_lastName" value="'%' + lastName + '%'"></bind>
>     select * from tbl_employee where last_name like #{_lastName}
> </select>
> ```
>
> ```java
> public List<Employee> getEmpsTestBind(Employee employee);
> ```

### 4.9 抽取重用的 sql 字段

> 1. 将要经常查询的列名，或插入用的列名抽取出来方便引用
>
> 2. 用 include 来引用抽取出的 sql
>
>    ```xml
>    <sql id="insertColumn">
>        <if test="_parameter.get('emps')[0].lastName!=null">
>            last_name,email,gender,d_id
>        </if>
>    </sql>
>    <insert id="addEmps">
>        insert into tbl_employee(
>        <!-- 引用外部定义的 sql -->
>        <include refid="insertColumn"></include>
>        ) values
>        <foreach collection="emps" item="emp" separator=','>
>            (#{emp.lastName},#{emp.email},#{emp.gender},#{emp.dept.id})
>        </foreach>
>    </insert>
>    ```
>
>    ```java
>    public void addEmps(@Param("emps")List<Employee> emps);
>    ```
>
> 3. 可以在 include 中定义 property，在 sql 标签中使用 ${} 取出值
>
>    ```xml
>    <sql id="insertColumn">
>        <if test="_parameter.get('emps')[0].lastName!=null">
>            last_name,email,gender,${did}
>        </if>
>    </sql>
>    <insert id="addEmps">
>        insert into tbl_employee(
>        <!-- 引用外部定义的 sql -->
>        <include refid="insertColumn">
>        	<property name="did" value="d_id"></property>
>        </include>
>        ) values
>        <foreach collection="emps" item="emp" separator=','>
>            (#{emp.lastName},#{emp.email},#{emp.gender},#{emp.dept.id})
>        </foreach>
>    </insert>
>    ```

---

## 5. 缓存

> Mybatis 系统中默认定义了两极缓存（map）：**一级缓存**（本地缓存），**二级缓存**（全局缓存）
>
> - 默认情况下，只有一级缓存（SqlSession 级别的缓存，也称为本地缓存）开启，同一个 SqlSession 拥有同一片缓存
> - 二级缓存需要手动开启和配置，它是基于 SqlSessionFactory 级别的缓存，同一个 SqlSessionFactory 拥有同一片缓存
> - 为了提高扩展性，Mybatis 定义了缓存接口 Cache。可以通过实现 Cache 接口来自定义二级缓存

### 5.1 一级缓存

> 与数据库同一次会话期间查询到的数据会放到本地缓存中，以后如果获取相同的数据，直接从缓存中拿，不会再去查询数据库
>
> ```java
> EmployeeMapper mapper = sqlSession.getMapper(EmployeeMapper.class);
> Employee empById1 = mapper.getEmpById(1);
> Employee empById2 = mapper.getEmpById(1);
> System.out.println(empById1==empById2); // true
> ```

#### 一级缓存失效的四种情况

> 没有使用到一级缓存的情况，即需要再向数据库发送查询
>
> 1. sqlSession 不同
>
>    ```java
>    EmployeeMapper mapper1 = sqlSession.getMapper(EmployeeMapper.class);
>    Employee empById1 = mapper1.getEmpById(1);
>    EmployeeMapper mapper2 = sqlSession.getMapper(EmployeeMapper.class);
>    Employee empById2 = mapper2.getEmpById(1);
>    System.out.println(empById1==empById2); // false
>    ```
>
> 2. sqlSession 相同，查询条件不同
>
> 3. sqlSession 相同，但两次查询间执行了增删改操作
>
> 4. sqlSession 相同，但手动清空了一级缓存：`sqlSession.clearCache()`

### 5.2 二级缓存

> 基于 SqlSessionFactory 级别的缓存，每个 nameSpace 对应一个二级缓存
>
> - 工作机制：
>
>   1. 一个会话查询一条数据，这个数据就会被放在当前会话的一级缓存中
>   2. 当会话关闭时，一级缓存中的数据会被保存到二级缓存中，新的会话查询信息就可以参照二级缓存
>   3. 不同 nameSpace 查出的数据会放在自己对应的缓存（map）中
>   4. 查询的数据默认先放在一级缓存中，只有**会话提交或关闭**后，一级缓存中的数据才会被转移到二级缓存中
>
> - 使用：
>
>   1. 开启全局二级缓存配置：`<setting name="cacheEnabled" value="true"/>`
>
>   2. 去 mapper.xml 中配置使用二级缓存
>
>      ```xml
>      <!--
>          eviction：缓存的回收策略
>              - LRU（默认）：最近最少使用的，移除最长时间不被使用的对象
>              - FIFO：先进先出
>              - SOFT：软引用，移除基于垃圾回收状态和软引用规则的对象
>              - WEAK：弱引用，更积极地移除基于垃圾收集器状态和软应用规则的对象
>          flushInterval：缓存刷新间隔，多长时间清空一次，默认不清空，设置一个毫秒值
>          readOnly：是否只读
>              - true：mybatis 认为所有从缓存中获取数据的操作都只是只读操作，不会修改数据，
>                      mybatis 为了加快获取速度，直接就会将数据在缓存中的引用交给用户，不安全但速度快
>              - false：mybatis 觉得获取的数据可能会被修改，mybatis 会利用序列化&反序列化技术克隆一份新的数据，安全但速度慢
>          size：缓存存放多少元素
>          type：指定自定义缓存的全类型，实现 Cache 接口
>      -->
>      <cache eviction="" flushInterval="" readOnly="" size="" type=""></cache>
>      ```
>
>   3. POJO 需要实现序列化接口
>
>      ```java
>      public class Employee implements Serializable {
>      ```

### 5.3 缓存相关的设置/属性

> - 全部设置 `cacheEnabled=false`：二级缓存关闭，一级缓存一直可用
> - 每个 select 标签都有 `userCache=true` 属性，代表开启或关闭二级缓存，一级缓存一直可用
> - 每个增删改标签的 `flushCache=true` 属性，默认为true，==执行完后会清空一级和二级缓存==
> - 查询标签也有 `flushCache=false` 默认 false，若改为 ture，则每次查询之后都会清空一级缓存，一级缓存失效
> - `sqlSession.clearCache()`：只会清空当前 session 的一级缓存
> - `localCacheScope`：本地缓存作用域
>   - 一级缓存SESSION：当前会话的所有数据保存在会话缓存中
>   - STATEMENT：可以禁用一级缓存

### 5.4 原理

<img src="..\img\image-20211228211912197.png" alt="image-20211228211912197" style="zoom:50%;" />



<img src="..\img\image-20211228213537333.png" alt="image-20211228213537333" style="zoom: 67%;" />

### 5.5 整合第三方缓存

> 1. 导入第三方缓存包
>
>    ```xml
>    <dependency>
>        <groupId>net.sf.ehcache</groupId>
>        <artifactId>ehcache-core</artifactId>
>        <version>2.6.11</version>
>    </dependency>
>    ```
>
> 2. 导入第三方缓存整合的适配包
>
>    ```xml
>    <dependency>
>        <groupId>org.mybatis.caches</groupId>
>        <artifactId>mybatis-ehcache</artifactId>
>        <version>1.0.3</version>
>    </dependency>
>    ```
>
> 3. 配置 ehcache.xml 文件
>
>    ```xml
>    <?xml version="1.0" encoding="UTF-8"?>
>    <ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
>             xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
>             updateCheck="false">
>        <!--
>           diskStore：为缓存路径，ehcache分为内存和磁盘两级，此属性定义磁盘的缓存位置。参数解释如下：
>           user.home – 用户主目录
>           user.dir  – 用户当前工作目录
>           java.io.tmpdir – 默认临时文件路径
>         -->
>        <diskStore path="D:/Tmp_EhCache"/>
>        <!--
>           defaultCache：默认缓存策略，当ehcache找不到定义的缓存时，则使用这个缓存策略。只能定义一个。
>         -->
>        <!--
>          name:缓存名称。
>          maxElementsInMemory:缓存最大数目
>          maxElementsOnDisk：硬盘最大缓存个数。
>          eternal:对象是否永久有效，一但设置了，timeout将不起作用。
>          overflowToDisk:是否保存到磁盘，当系统当机时
>          timeToIdleSeconds:设置对象在失效前的允许闲置时间（单位：秒）。仅当eternal=false对象不是永久有效时使用，可选属性，默认值是0，也就是可闲置时间无穷大。
>          timeToLiveSeconds:设置对象在失效前允许存活时间（单位：秒）。最大时间介于创建时间和失效时间之间。仅当eternal=false对象不是永久有效时使用，默认是0.，也就是对象存活时间无穷大。
>          diskPersistent：是否缓存虚拟机重启期数据 Whether the disk store persists between restarts of the Virtual Machine. The default value is false.
>          diskSpoolBufferSizeMB：这个参数设置DiskStore（磁盘缓存）的缓存区大小。默认是30MB。每个Cache都应该有自己的一个缓冲区。
>          diskExpiryThreadIntervalSeconds：磁盘失效线程运行时间间隔，默认是120秒。
>          memoryStoreEvictionPolicy：当达到maxElementsInMemory限制时，Ehcache将会根据指定的策略去清理内存。默认策略是LRU（最近最少使用）。你可以设置为FIFO（先进先出）或是LFU（较少使用）。
>          clearOnFlush：内存数量最大时是否清除。
>          memoryStoreEvictionPolicy:可选策略有：LRU（最近最少使用，默认策略）、FIFO（先进先出）、LFU（最少访问次数）。
>          FIFO，first in first out，这个是大家最熟的，先进先出。
>          LFU， Less Frequently Used，就是上面例子中使用的策略，直白一点就是讲一直以来最少被使用的。如上面所讲，缓存的元素有一个hit属性，hit值最小的将会被清出缓存。
>          LRU，Least Recently Used，最近最少使用的，缓存的元素有一个时间戳，当缓存容量满了，而又需要腾出地方来缓存新的元素的时候，那么现有缓存元素中时间戳离当前时间最远的元素将被清出缓存。
>       -->
>        <defaultCache
>                      eternal="false"
>                      maxElementsInMemory="10000"
>                      overflowToDisk="true"
>                      diskPersistent="false"
>                      timeToIdleSeconds="1800"
>                      timeToLiveSeconds="259200"
>                      memoryStoreEvictionPolicy="LRU"/>
>        
>    </ehcache>
>    ```
>
> 4. 在 mapper.xml 中使用自定义缓存
>
>    ```xml
>    <cache type="org.mybatis.caches.ehcache.EhcacheCache"/>
>    ```
>
>    或者：
>
>    ```xml
>    <!-- 指定和哪个名称空间下的缓存一样 -->
>    <cache-ref namespace="com.mybatis.dao.EmployeeMapper"/>
>    ```

---

## 6. 整合 Spring Boot 2

> spring MVC 的整合可以参考：[链接](https://github.com/mybatis/jpetstore-6/blob/master/src/main/webapp/WEB-INF/applicationContext.xml)

### 6.1 Spring Boot 2 导入流程

> 1. 引入 dependency
>
>    ```xml
>    <dependency>
>        <groupId>org.springframework.boot</groupId>
>        <artifactId>spring-boot-starter-web</artifactId>
>    </dependency>
>    <dependency>
>        <groupId>org.mybatis.spring.boot</groupId>
>        <artifactId>mybatis-spring-boot-starter</artifactId>
>        <version>2.2.0</version>
>    </dependency>
>    <dependency>
>        <groupId>mysql</groupId>
>        <artifactId>mysql-connector-java</artifactId>
>        <scope>runtime</scope>
>    </dependency>
>    <dependency>
>        <groupId>org.springframework.boot</groupId>
>        <artifactId>spring-boot-starter-test</artifactId>
>        <scope>test</scope>
>    </dependency>
>    ```
>
> 2. 配置 application.properties
>
>    ```properties
>    spring.datasource.username=root
>    spring.datasource.password=root
>    spring.datasource.url=jdbc:mysql://localhost:3306/mybatis?allowPublicKeyRetrieval=true
>    spring.datasource.driver-class-name=com.mysql.jdbc.Driver
>    
>    // 指定 mapper.xml 文件地址
>    mybatis.mapper-locations=classpath:mybatis.mapper/*.xml
>    
>    mybatis.configuration.map-underscore-to-camel-case=true
>    // 也可指定 config 文件地址，在 config 文件中声明mybatis配置，但此 application.properties 文件下的 configuration 属性会失效
>    mybatis.config-location=classpath:mybatis/mybatis-config.xml
>    
>    logging.level.com.example.integratemybatis.dao=debug
>    ```
>
> 3. 写 mapper 接口并标注 `@Mapper` 注解
>
>    ```java
>    @Mapper
>    public interface EmployeeMapper {
>        public Employee getEmp(Integer id);
>    }
>    ```
>
>    也可以直接在主类上标注 `@MapperScan`，此时就不用在接口上标注 @Mapper 注解
>
>    ```java
>    @MapperScan("com.example.mapper")
>    ```
>
> 4. 写 mapper.xml 或在 mapper 接口上使用 CRUD 注解
>
>    ```xml
>    <?xml version="1.0" encoding="UTF-8" ?>
>    <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
>    <mapper namespace="com.example.integratemybatis.dao.EmployeeMapper">
>        <select id="getEmp" resultType="com.example.integratemybatis.bean.Employee">
>            select * from tbl_employee where id=#{id}
>        </select>
>    </mapper>
>    ```

### 6.2 开启事务

> - 标注 `@EnableTransactionManagement` 开启基于注解的事务管理功能
>
>   ```java
>   @SpringBootApplication
>   @EnableTransactionManagement
>   public class IntegrateMybatisApplication {
>   ```
>
> - 给方法上标注 `@Transactional` 表示当前方法是一个事务
>
>   ```java
>   @RequestMapping("/emp")
>   @Transactional
>   public void getEmployee() {
>       employeeMapper.insertEmp("test");
>       int i = 3 / 0;
>   }
>   ```
>
> - JDBC 的 autoconfigure 会自动给容器中添加 DataSource 和 DataSourceTransactionManager，不需手动配置

---

## 7. 逆向工程

>     简称 MBG，是一个专门为 MyBatis 框架使用者制定的代码生成器，可以快速地根据表生成对应的映射文件，接口，以及 Bean 类，支持基本的增删改查，以及 QBC 风格的条件查询。但是表连接，存储过程等复杂 sql 的定义需要我们手工编写
>
>   [参考](https://mybatis.org/generator/index.html)

### 1. 引入 jar 包

```xml
<dependency>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-core</artifactId>
    <version>1.4.0</version>
</dependency>
```

### 2. 编写配置文件

**generatorConfig.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>

    <!--
    MyBatis3Simple：生成简单版的CRUD
    Mybatis3：生成豪华版的
    -->
    <context id="DB2Tables" targetRuntime="MyBatis3">
        <!-- jdbcConnection：指定如何连接到数据库 -->
        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                        connectionURL="jdbc:mysql://localhost:3306/mybatis?allowPublicKeyRetrieval=true"
                        userId="root"
                        password="root">
        </jdbcConnection>

        <javaTypeResolver >
            <property name="forceBigDecimals" value="false" />
        </javaTypeResolver>

        <!--
        指定 javaBean 的生成策略
            - targetPackage：目标包名
            - targetProject：目标工程
        -->
        <javaModelGenerator targetPackage="com.example.integratemybatis.bean" targetProject=".\src\main\java">
            <property name="enableSubPackages" value="true" />
            <property name="trimStrings" value="true" />
        </javaModelGenerator>

        <!--
        Sql 映射策略
        -->
        <sqlMapGenerator targetPackage="mybatis.mapper"  targetProject=".\src\main\resources">
            <property name="enableSubPackages" value="true" />
        </sqlMapGenerator>

        <!--
        指定 mapper 接口的位置
        -->
        <javaClientGenerator type="XMLMAPPER" targetPackage="com.example.integratemybatis.dao"  targetProject=".\src\main\java">
            <property name="enableSubPackages" value="true" />
        </javaClientGenerator>

        <!--
        指定要逆向分析哪些表，根据表创建 javaBean
        -->
        <table tableName="tbl_dept" domainObjectName="Department"></table>
        <table tableName="tbl_employee" domainObjectName="Employee"></table>

    </context>
</generatorConfiguration>
```

### 3. running

> 1. 添加 maven plugin
>
>    ```xml
>    <plugin>
>        <groupId>org.mybatis.generator</groupId>
>        <artifactId>mybatis-generator-maven-plugin</artifactId>
>        <version>1.4.0</version>
>        <!-- 不写会产生 classloader 问题 -->
>        <dependencies>
>            <dependency>
>                <groupId>mysql</groupId>
>                <artifactId>mysql-connector-java</artifactId>
>                <version>8.0.27</version>
>            </dependency>
>        </dependencies>
>    </plugin>
>    ```
>
> 2. 运行指令：`mvn mybatis-generator:generate`

### 4. 测试

```java
@SpringBootTest()
class IntegrateMybatisApplicationTests {

	@Autowired
	EmployeeMapper employeeMapper;

	@Test
	void contextLoads() {
		// xxxExample 就是封装查询条件的
		// 1. 查询所有
		List<Employee> employees1 = employeeMapper.selectByExample(null);

		// 2. 查询员工名字中有 a 字母的，和性别是 0，或者 email 中 s 有
		EmployeeExample example = new EmployeeExample();
		// 创建一个 Criteria，构造拼装条件
		EmployeeExample.Criteria criteria1 = example.createCriteria();
		criteria1.andLastNameLike("%a%");
		criteria1.andGenderEqualTo("0");

		EmployeeExample.Criteria criteria2 = example.createCriteria();
		criteria2.andEmailLike("%e%");
		example.or(criteria2);

		List<Employee> employees2 = employeeMapper.selectByExample(example);
		employees2.forEach(a -> System.out.println(a.getId()));

	}

}
```

---

## 8. 运行原理

![image-20211229205427060](..\img\image-20211229205427060.png)

> **1. 获取 sqlSessionFactory 对象**
>
> - MybatisAutoConfiguration.class - sqlSessionFactory() - factory.getObject() -> SqlSessionFactoryBean.class - buildSqlSessionFactory() - sqlSessionFactoryBuilder.build(targetConfiguration)
> - 把配置文件的信息jie解析并保存到 Configuration 对象中，返回包含了 Configuration 的 DefaultSqlSession 对象
>
> - 注意：MapperStatement 代表一个增删改查的详细信息
>
> <img src="..\img\image-20211229211920563.png" style="zoom:67%;" />
>
> **2. 获取 sqlSession 对象**
>
> - 返回一个 DefaultSqlSession 对象，包含 Executor 和 Configuration
>
> <img src="..\img\image-20211229213729756.png" alt="image-20211229213729756" style="zoom:67%;" />
>
> **3. 获取接口的实现类对象（MapperProxy）**
>
> - 使用 MapperProxyFactory 创建一个 MapperProxy 代理对象，包含了 DefaultSqlSession
>
> <img src="..\img\image-20211229214800638.png" alt="image-20211229214800638" style="zoom:67%;" />
>
> 
>
> **4. 执行增删改查方法**
>
> ==流程==
>
> ```mermaid
> sequenceDiagram
> 	MapperProxy ->> MapperMethod: invoke()
> 	MapperMethod ->> MapperMethod: 判断增删改查类型
> 	MapperMethod ->> MapperMethod: 包装参数为一个 map 或者直接返回
> 	MapperMethod ->> DefaultSqlSession: sqlSession.selectOne()
> 	DefaultSqlSession ->> DefaultSqlSession: selectList()
> 	DefaultSqlSession ->> DefaultSqlSession: 获取 MappedStatement
> 	DefaultSqlSession ->> Executor: executor.query()
> 	Executor ->> Executor: 获取 BoundSql，它代表 sql 语句的详细信息
> 	Executor ->> SimpleExecutor: executor.query()
> 	SimpleExecutor ->> SimpleExecutor: 查看本地缓存是否有数据，没有就调用 queryFromDatabase，查出以后也会保存在本地缓存
> 	SimpleExecutor ->> BaseExecutor: doQuery()
> 	BaseExecutor ->> StatementHandler: 创建 StatementHandler 对象，PreparedStatementHandler
> 	StatementHandler ->> StatementHandler: (StatementHandler) interceptorChain.pluginAll(statementHandler)
> 	StatementHandler ->> StatementHandler: 创建 ParameterHandler：interceptorChain.plugin(parameterHandler)
> 	StatementHandler ->> StatementHandler: 创建 ResultSetHandler： interceptorChain.plugin(resultSetHandler)
> 	StatementHandler ->> StatementHandler: 预编译 sql 产生 PreparedStatement 对象
> 	StatementHandler ->> StatementHandler: 调用 ParameterHandler 设置参数
> 	StatementHandler ->> StatementHandler: 调用 TypeHandler 给 sql 预编译设置参数
> 	StatementHandler ->> StatementHandler: 查出数据使用 ResultSetHandler 处理结果：使用 TypeHandler 获取 value 值
> 	StatementHandler ->> DefaultSqlSession: 连接关闭等操作
> 	DefaultSqlSession ->> MapperMethod: 返回 list 第一个
> 	
> ```
>
> 
>
> - StatementHandler：处理 sql 语句预编译，设置参数等相关工作
> - ParameterHandler：设置预编译参数
> - ResultHandler：处理结果集
> - TypeHandler：在整个过程中，进行数据库类型和 javaBean 类型的映射
>
> ```mermaid
> graph TB
> 	代理对象 --> DefaultSqlSession
> 	DefaultSqlSession --> Executor
> 	Executor --> StatementHandler
> 	StatementHandler --设置参数--> ParameterHandler
> 	StatementHandler --处理结果--> ResultSetHandler
> 	ParameterHandler --> TypeHandler
> 	ResultSetHandler --> TypeHandler
> 	TypeHandler --> JDBC:Statument:PreparedStatement
> 
> ```
>
> ==总结==
>
> 1. 根据配置文件（全局，sql 映射）初始化出 Configuration 对象
> 2. 创建一个 DefaultSqlSession 对象，厘米那包含 Configuration 和 Executor（根据全局配置文件中的 defaultExecutorType 创建出对应的 Executor）
> 3. DefaultSqlSession.getMapper()：拿到 Mapper 接口对应的 MapperProxy（包含 DefaultSqlSession）
> 4. 执行增删改查方法：
>    1. 调用 DefaultSqlSession 的增删改查（Executor）
>    2. 创建一个 StatementHandler 对象同时也会创建出 ParameterHandler 和 ResultSetHandler
>    3. 调用 StatementHandler 的预编译参数以及设置参数值
>       - 调用 ParameterHandler 给 sql 设置参数
>    4. 调用 StatementHandler 的增删改查方法
>    5. 使用 ResultSetHandler 封装结果
>
> ==注意==
>
> - 四大对象每个创建的时候都有会调用 interceptorChain.pluginAll() 方法

---

## 9. 插件开发

> - Mybatis 在四大对象的创建过程中，都会有插件进行接入，插件可以利用动态代理机制一层层的包装目标对象，而实现在目标对象执行目标方法之前进行拦截的效果
> - Mybatis 允许在已映射语句执行过程中的某一点进行拦截调用
> - 默认情况下，Mybatis 允许使用插件来拦截的方法调用包括：
>   - Executor
>   - ParameterHandler
>   - ResultSetHandler
>   - StatementHandler
> - ==原理==
>   1. 四大对象在创建时会调用 interceptorChain.pluginAll()
>   2. 获取到所有的 Interceptor，调用 interceptor.plugin(target)，返回包装后的对象
>   3. 我们可以使用插件为目标对象创建一个代理对象 —— AOP，由代理对象拦截到四大对象的每一步执行

### 编写插件

> 1. 编写 Interceptor 的实现类
>
>    ```java
>    // 插件签名：告诉 myBatis 拦截哪个对象的哪个方法
>    @Intercepts(
>            {@Signature(type= StatementHandler.class, method="parameterize", args= Statement.class)}
>    )
>    public class MyInterceptor implements Interceptor {
>        /**
>         * 拦截目标方法的执行
>         */
>        @Override
>        public Object intercept(Invocation invocation) throws Throwable {
>            // 执行目标方法前
>            beforeProceed();
>            // 执行目标方法
>            Object proceed = invocation.proceed();
>            // 执行目标方法后
>            afterProceed();
>            
>            return proceed;
>        }
>    
>        /**
>         * 包装目标对象：为目标对象创建一个代理对象
>         * 在调用 pluginAll() 时被调用
>         */
>        @Override
>        public Object plugin(Object target) {
>            // 借助 Plugin 的 wrap 方法来包装
>            Object wrap = Plugin.wrap(target, this);
>            // 返回创建的动态代理对象
>            return wrap;
>        }
>    
>        /**
>         * 将插件注册时的 Property 属性设置进来
>         */
>        @Override
>        public void setProperties(Properties properties) {
>            System.out.println("插件配置的信息" + properties);
>        }
>    }
>    ```
>
> 2. 使用 `＠Intercepts` 注解完成插件签名
>
> 3. 将写好的插件注册到全局配置文件中
>
>    ```xml
>    <?xml version="1.0" encoding="UTF-8" ?>
>    <!DOCTYPE configuration
>            PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
>            "http://mybatis.org/dtd/mybatis-3-config.dtd">
>    <configuration>
>        <plugins>
>            <plugin interceptor="com.example.integratemybatis.MyInterceptor">
>                <!-- 会传入 setProperties() 方法 -->
>                <property name="name" value="root"/>
>                <property name="password" value="root"/>
>            </plugin>
>        </plugins>
>    </configuration>
>    ```

### 多个插件运行流程

>   在配置文件中先配置的插件先被创建，并且包装时后面的插件会包装前面插件包装后的对象奥，会产生层层代理，执行目标方法时，按照逆向执行拦截方法
>
> <img src="..\img\image-20220101130341858.png" alt="image-20220101130341858" style="zoom: 50%;" />

### 定制插件

> ```java
> @Override
> public Object intercept(Invocation invocation) throws Throwable {
> 
>     // 拿到 StatementHandler -> ParameterHandler -> parameterObject（存储了 sql 参数信息）
>     Object target = invocation.getTarget();
>     // 拿到 target 的元数据
>     MetaObject metaObject = SystemMetaObject.forObject(target);
>     Object value = metaObject.getValue("parameterHandler.parameterObject");
>     System.out.println("sql 语句的参数是" + value);
>     // 修改参数
>     metaObject.setValue("parameterHandler.parameterObject", new Object());
> 
>     // 执行目标方法
>     Object proceed = invocation.proceed();
>     return proceed;
> }
> ```

---

## 10. 扩展

### 10.1 PageHelper 插件

> [参考](https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/HowToUse.md)
>
> - 导入
>
>   ```xml
>   <dependency>
>       <groupId>com.github.pagehelper</groupId>
>       <artifactId>pagehelper-spring-boot-starter</artifactId>
>       <version>1.4.1</version>
>   </dependency>
>   ```
>
> - 获取第一页记录，每一页 3 条记录
>
>   ```java
>   PageHelper.startPage(1, 3);
>   List<Employee> emps = employeeMapper.getEmps();
>   ```
>
> - 获取详细信息
>
>   ```java
>   Page<Object> page = PageHelper.startPage(1, 3);
>   List<Employee> emps = employeeMapper.getEmps();
>   System.out.println("当前页数：" + page.getPageNum());
>   System.out.println("总记录数：" + page.getTotal());
>   System.out.println("每页记录数：" + page.getPageSize());
>   System.out.println("总页码：" + page.getPages());
>   ```
>
>   ```java
>   // 使用 PageInfo 封装信息
>   Page<Object> page = PageHelper.startPage(1, 3);
>   List<Employee> emps = employeeMapper.getEmps();
>   PageInfo<Employee> info = new PageInfo<>(emps);
>   System.out.println("当前页数：" + info.getPageNum());
>   System.out.println("总记录数：" + info.getTotal());
>   System.out.println("每页记录数：" + info.getPageSize());
>   System.out.println("总页码：" + info.getPages());
>   System.out.println("是否第一页：" + info.isIsFirstPage());
>   System.out.println("是否是最后一页：" + info.isIsLastPage());
>   ```
>
>   ```java
>   // 传入连续显示多少页
>   PageInfo<Employee> info = new PageInfo<>(emps, 3);
>   int[] navigatepageNums = info.getNavigatepageNums();
>   // 当前第三页：2 3 4，当前第五页：4 5 6
>   Arrays.stream(navigatepageNums).forEach(a -> System.out.println("连续显示的页码：" + a));
>   ```

### 10.2 批量操作

> ```properties
> # url开启数据库支持 batch 操作
> &rewriteBatchedStatements=true
> ```
>
> ```java
> @Bean
> public SqlSession batchSqlSession(SqlSessionFactory sqlSessionFactory) {
>  return sqlSessionFactory.openSession(ExecutorType.BATCH);
> }
> ```
>
> ```java
> @Autowired
> SqlSession batchSqlSession;
> 
> @Test
> void testBatch() {
>  try{
>      EmployeeMapper mapper = batchSqlSession.getMapper(EmployeeMapper.class);
>      // 非批量：(预编译 sql -> 设置参数 -> 执行) * n
>      // 批量： 预编译 sql * 1 -> 设置参数 * n -> 执行 * 1
>      for (int i = 0; i < 1000; i++) {
>          mapper.addEmp(new Employee());
>      }
>  } finally {
>      batchSqlSession.close();
>  }
> }
> ```

### 10.3 存储过程

> 1. Oracle 中创建一个带游标的存储过程
>
>    ![image-20220101154808387](..\img\image-20220101154808387.png)
>
> 2. mybatis 调用存储过程
>
>    ```java
>    void getPageByProcedure(Pages page);
>    
>    public class Pages {
>        private int start;
>        private int end;
>        private int count;
>        private List<Employee> emps;
>    }
>    ```
>
>    ```xml
>     <!--
>      1. 使用 select 标签定义存储过程
>      2. 使用 CALLABLE statementType 表示调用存储过程，默认 prepare
>      3. {call procedureName(param)}
>      -->
>      <select id="getPageByProcedure" statementType="CALLABLE" databaseId="oracle">
>        {call hello_test(
>            #{start,mode=IN,jdbcType=INTEGER},
>            #{end,mode=IN,jdbcType=INTEGER},
>            #{count,mode=OUT,jdbcType=INTEGER},
>            #{emps,mode=OUT,jdbcType=CURSOR,javaType=ResultSet,resultMap=pageEmp}
>          )}
>      </select>
>      <resultMap id="pageEmp" type="com.example.integratemybatis.bean.Employee">
>        <id column="EMPLOYEE_ID" property="id"></id>
>        <result column="LAST_NAME" property="lastName"></result>
>        <result column="EMAIL" property="email"></result>
>      </resultMap>
>    ```
>
> 3. 测试
>
>    ```java
>    @Test
>    public void testProcedure() {
>        Pages page = new Pages();
>        page.setStart(1);
>        page.setEnd(5);
>        employeeMapper.getPageByProcedure(page);
>        System.out.println(page.getCount());
>    }
>    ```

### 10.4 自定义类型处理器

> 1. 默认处理枚举的情况
>
>    - 默认 mybatis 在处理枚举对象时保存的是枚举的名字 -> EnumTypeHandler.class
>
>    - 改变使用 EnumOrdinalTypeHandler，即使用索引作为值
>
>      ```xml
>      <?xml version="1.0" encoding="UTF-8" ?>
>      <!DOCTYPE configuration
>              PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
>              "http://mybatis.org/dtd/mybatis-3-config.dtd">
>      <configuration>
>          <typeHandlers>
>              <typeHandler handler="org.apache.ibatis.type.EnumOrdinalTypeHandler" javaType="com.example.integratemybatis.bean.EmpStatus"></typeHandler>
>          </typeHandlers>
>      </configuration>
>      ```
>
> 2. 自定义类型处理器
>
>    ```java
>    /**
>     * 实现 TypeHandler 接口 或者继承 BaseTypeHandler
>     */
>    public class MyEnumEmpStatusTypeHandler implements TypeHandler<EmpStatus> {
>        /**
>         * 定义如何保存到数据库中
>         */
>        @Override
>        public void setParameter(PreparedStatement ps, int i, EmpStatus parameter, JdbcType jdbcType) throws SQLException {
>            ps.setString(i, parameter.getCode().toString());
>        }
>                                     
>        @Override
>        public EmpStatus getResult(ResultSet rs, String columnName) throws SQLException {
>            // 需要根据从数据库中拿到的枚举的状态码返回一个枚举对象
>            int code = rs.getInt(columnName);
>            EmpStatus empStatus = Employee.getEmpStatus(code);
>            return empStatus;
>                                     
>        }
>                                     
>        @Override
>        public EmpStatus getResult(ResultSet rs, int columnIndex) throws SQLException {
>            int code = rs.getInt(columnIndex);
>            EmpStatus empStatus = Employee.getEmpStatus(code);
>            return empStatus;
>        }
>                                     
>        @Override
>        public EmpStatus getResult(CallableStatement cs, int columnIndex) throws SQLException {
>            int code = cs.getInt(columnIndex);
>            EmpStatus empStatus = Employee.getEmpStatus(code);
>            return empStatus;
>        }
>    }
>    ```
>
>    ```java
>    /**
>     * 希望数据库保存的是100， 200状态码
>     */
>    public enum EmpStatus {
>                                     
>        LOGIN(100,"用户登录"),LOGOUT(200, "用户登出"),REMOVE(300, "用户移除");
>                                     
>        private Integer code;
>        private String message;
>        private EmpStatus(Integer code, String message) {
>            this.code = code;
>            this.message = message;
>        }
>                                     
>        public Integer getCode() {
>            return code;
>        }
>                                     
>        public void setCode(Integer code) {
>            this.code = code;
>        }
>                                     
>        public String getMessage() {
>            return message;
>        }
>                                     
>        public void setMessage(String message) {
>            this.message = message;
>        }
>    }
>    ```
>
>    ```java
>    Employee.class
>                                     
>    public EmpStatus empStatus;
>                                     
>    public EmpStatus getEmpStatus() {
>        return empStatus;
>    }
>                                     
>    public void setEmpStatus(EmpStatus empStatus) {
>        this.empStatus = empStatus;
>    }
>                                     
>    /**
>         * 根据状态码返回枚举对象
>         */
>    public static EmpStatus getEmpStatus(Integer code) {
>        switch (code) {
>            case 100:
>                return EmpStatus.LOGIN;
>            case 200:
>                return EmpStatus.LOGOUT;
>            case 300:
>                return EmpStatus.REMOVE;
>            default:
>                return EmpStatus.LOGOUT;
>        }
>    }
>    ```
>
>    配置：
>
>    1. 在 mybatis 配置文件中设置
>
>       ```xml
>       <?xml version="1.0" encoding="UTF-8" ?>
>       <!DOCTYPE configuration
>               PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
>               "http://mybatis.org/dtd/mybatis-3-config.dtd">
>       <configuration>
>           <typeHandlers>
>               <typeHandler handler="com.example.integratemybatis.bean.MyEnumEmpStatusTypeHandler" javaType="com.example.integratemybatis.bean.EmpStatus"></typeHandler>
>               <!--
>               保存时，也可以在mapper.xml中处理某个字段时告诉 mybatis 用什么类型处理器：#{empStatus, typeHandler=xxx}
>               查询时，可以自定义一个 resultMap，指定：<result column="empStatus" property="empStatus" typeHandler=xxx>
>               注意：如果在参数位置修改 TypeHandler，应该保证存储数据和查询数据使用的 TypeHandler 一致
>                -->
>           </typeHandlers>
>       </configuration>
>       ```
>
>    2. 在 springboot 的 yml 配置文件中设置类型处理器所在的包名
>
>       ```yaml
>       mybatis:
>         type-handlers-package: com.xxx.handler
>       ```
>
>       - 需要指定处理的类型
>
>         ```java
>         @MappedTypes(value = {EmpStatus.class})
>         public class MyEnumEmpStatusTypeHandler implements TypeHandler<EmpStatus> {
>         ```
