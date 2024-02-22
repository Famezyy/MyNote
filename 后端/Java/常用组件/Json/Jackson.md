# Jackson

## 1.快速上手

导入依赖

~~~xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.13.3</version>
</dependency>
~~~

序列化

```java
ObjectMapper objectMapper = new ObjectMapper();
User zhao = new User();
zhao.setName("zhao");
String string = objectMapper.writeValueAsString(zhao);
System.out.println(string); // {"name":"zhao","age":null}
```

反序列化

```java
ObjectMapper objectMapper = new ObjectMapper();
String user = "{\"name\":\"zhao\"}"; // 属性必须用双引号，且总是会返回一个对象，若为 {}，则返回一个空对象
User zhao = objectMapper.readValue(user, User.class);
System.out.println(zhao); // User(name=zhao, age=null)
```

## 2.配置

### 2.1 setSerializationInclustion(JsonInclude)

|            JsonInclude            |              作用              |
| :-------------------------------: | :----------------------------: |
|  `JsonInclude.Include.NON_NULL`   | 序列化不包含 null 值（非默认） |
| `JsonInclude.Include.NON_DEFAULT` |  排除 POJO 中具有默认值的字段  |
|   `JsonInclude.Include.ALWAYS`    |        总是包含所有属性        |
|  `JsonInclude.Include.NON_EMPTY`  |   属性为非空（""）且非 null    |
|  `JsonInclude.Include.NON_NULL`   |          属性非 null           |

#### 1.序列化不包含null值（默认包含）

`objectMapper.setSerializationInclusion(JsonInclude.Include.NON_NULL)`

```java
ObjectMapper objectMapper = new ObjectMapper();
objectMapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
User zhao = new User();
zhao.setName("zhao");
String string = objectMapper.writeValueAsString(zhao);
System.out.println(string); // {"name":"zhao"}
```

#### 2.排除具有POJO默认值的属性

`ObjectMapper.setDefaultPropertyInclusion(JsonInclude.Include.NON_DEFAULT)`，作用同`@JsonInclude(JsonInclude.Include.NON_DEFAULT)`注解在字段上。

### 2.2 configure(Feature, boolean)

> 也可以使用`enable(Feature)`和`disable(Feature)`。

|                           Feature                           |                             作用                             |
| :---------------------------------------------------------: | :----------------------------------------------------------: |
|            `SerializationFeature.INDENT_OUTPUT`             |               对输出缩进，美化输出，默认 false               |
|         `SerializationFeature.FAIL_ON_EMPTY_BEAMS`          | 对于空对象（没有赋值），如果没有注解`@JsonSerialize`则会报错，默认 true |
|      `SerializationFeature.WRITE_DATES_AS_TIMESTAMPS`       | 将日期序列化为时间戳，时区信息会丢失，建议设置 false 禁止<br />默认为 true 时：`{"userName":null,"age":0,"zonedDateTime":1656588102.277000000,"localDateTime":[2022,6,30,19,21,42,277000000]}` |
|       `SerializationFeature.WRITE_DATES_WITH_ZONE_ID`       | 在类型本身包含时区信息的情况下，将 ZoneID 序列化（[Asia/Shanghai]），默认 false |
|     `DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES`     |           反序列化不允许存在没有的字段，默认 true            |
| `DeserializationFeature.ADJUST_DATES_TO_CONTEXT_TIME_ZONE`  | 反系列化日期采用默认上下文的 timeZone，默认 true<br />**为 true 时，反序列化 ZoneDateTime 会自动转换为 UTC 时区** |
| `DeserializationFeature.ACCEPT_EMPTY_STRING_AS_NULL_OBJECT` | 反序列化时，如果对象中的类成员变量的值是空字符串`""`，那么该类属性设置为 null，默认 false，即如果是空字符串，则反序列化报错 |
|         `MapperFeature.PROPAGATE_TRANSIENT_MARKER`          |            忽略 transient 修饰的属性，默认 false             |

#### 1.序列化美化输出

`objectMapper.configure(SerializationFeature.INDENT_OUTPUT, true)`

```java
ObjectMapper objectMapper = new ObjectMapper();
objectMapper.configure(SerializationFeature.INDENT_OUTPUT, true);
User zhao = new User();
zhao.setName("zhao");
String string = objectMapper.writeValueAsString(zhao);
System.out.println(string);
```

```bash
{
  "name" : "zhao"
}
```

#### 2.反序列化允许存在没有的字段

反序列化时若存在实体类中没有的属性，则会==报错==：

```java
ObjectMapper objectMapper = new ObjectMapper();
// 实体类 User 中没有 size 这个属性
String user = "{\"name\":\"zhao\",\"size\":\"15\"}";
User zhao = objectMapper.readValue(user, User.class);
System.out.println(zhao);
```

```bash
Exception in thread "main" com.fasterxml.jackson.databind.exc.UnrecognizedPropertyException: Unrecognized field "size" (class User), not marked as ignorable (2 known properties: "name", "age"])
```

配置允许存在实体类中没有的字段

`objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)`

或者

`objectMapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)`

```java
ObjectMapper objectMapper = new ObjectMapper();
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
// objectMapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
String user = "{\"name\":\"zhao\",\"size\":\"15\"}";
User zhao = objectMapper.readValue(user, User.class);
System.out.println(zhao);
```

### 2.3 setPropertyNamingStrategy(PropertyNamingStrategies)

#### 1.驼峰转下划线

`objectMapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE)`

```java
ObjectMapper objectMapper = new ObjectMapper();
objectMapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);

User user1 = new User("zhao", 16);
System.out.println(objectMapper.writeValueAsString(user1)); // {"user_name":"zhao","cur_age":16}

// 若实体类和字符串的名称一样则正常匹配，若不一样
// 实体类的驼峰 <-> 字符串的下划线
String string = "{\"user_name\":\"zhao\", \"cur_age\":\"16\"}";
User user2 = objectMapper.readValue(string, User.class); 
System.out.println(user2); // User(userName=zhao, cur_age=16)
```

### 2.4 setVisibility(PropertyAccessor, Visibility)

```java
// 仅属性会被序列化和反序列化
objectMapper.setVisibility(PropertyAccessor.ALL, Visibility.NONE);
objectMapper.setVisibility(PropertyAccessor.FIELD, Visibility.ANY);
```

### 2.5 注解

#### 1.序列化不包含null值（默认包含）

`@JsonInclude`

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public class User {
```

#### 2.排除具有POJO默认值的属性

如果在类级别使用`@JsonInclude(JsonInclude.Include.NON_DEFAULT)`，则将排除字段的默认值。这是通过使用无参数构造函数创建 POJO 实例并比较属性值来完成的。

例如：

```java
@JsonInclude(JsonInclude.Include.NON_DEFAULT)
public class Employee {
     private String name;
     private String dept;
     private Integer salary;
     private boolean fullTime;
     private List<String> phones;
     private Date dateOfBirth;
}

public class ExampleMain {
    public static void main(String[] args) throws IOException {
        Employee employee = new Employee();
        employee.setName("Trish");
        employee.setFullTime(false); // 会被排除
        employee.setPhones(new ArrayList<>()); // 不排除空集合
        employee.setSalary(Integer.valueOf(0)); // 不排除包装类的默认值
        employee.setDateOfBirth(new Date(0)); // 不排除 0 毫秒的日期

        ObjectMapper om = new ObjectMapper();
        String jsonString = om.writeValueAsString(employee);
        System.out.println(jsonString); // {"name":"Trish","salary":0,"phones":[],"dateOfBirth":0}
    }
}
```

如果在属性级别使用`@JsonInclude(JsonInclude.Include.NON_DEFAULT)`，在两种情况下：

- 所有被视为“空”的值（根据 NON_EMPTY） 被排除在外
- 基本/包装类型默认值被排除
- 排除毫秒值为“ 0L”的日期/时间值

```java
public class Employee2 {
    @JsonInclude(JsonInclude.Include.NON_DEFAULT)
    private String name;
    @JsonInclude(JsonInclude.Include.NON_DEFAULT)
    private String dept;
    @JsonInclude(JsonInclude.Include.NON_DEFAULT)
    private Integer salary;
    @JsonInclude(JsonInclude.Include.NON_DEFAULT)
    private boolean fullTime;
    @JsonInclude(JsonInclude.Include.NON_DEFAULT)
    private List<String> phones;
    @JsonInclude(JsonInclude.Include.NON_DEFAULT)
    private Date dateOfBirth;
}

public class ExampleMain2 {
    public static void main(String[] args) throws IOException {
        Employee2 employee = new Employee2();
        employee.setName("Trish");
        employee.setFullTime(false);
        employee.setPhones(new ArrayList<>());
        employee.setSalary(Integer.valueOf(0));
        employee.setDateOfBirth(new Date(0));

        ObjectMapper om = new ObjectMapper();
        String jsonString = om.writeValueAsString(employee);
        System.out.println(jsonString); // {"name":"Trish"}
    }
}
```

#### 3.指定字段的序列化名称

通过`@JsonProperty("name")`指定后，则属性名不再起作用：

```Java
public class User {
    @JsonProperty("name")
    private String userName;
    private Integer curAge;
}
```

```java
ObjectMapper objectMapper = new ObjectMapper();

User user1 = new User("zhao", 16);
System.out.println(objectMapper.writeValueAsString(user1)); // {"curAge":16,"name":"zhao"}

// String string = "{\"userName\":\"zhao\", \"curAge\":\"16\"}"; 报错！
String string = "{\"name\":\"zhao\", \"curAge\":\"16\"}";
User user2 = objectMapper.readValue(string, User.class);
System.out.println(user2); // User(userName=zhao, cur_age=16)
```

#### 4.忽略指定字段

使用`@JsonIgnore`后，序列化和反序列化都会忽略该属性：

```java
public class User {
    @JsonIgnore
    private String userName;
    private Integer curAge;
}
```

```java
ObjectMapper objectMapper = new ObjectMapper();

        User user1 = new User("zhao", 16);
        System.out.println(objectMapper.writeValueAsString(user1)); // {"curAge":16}

        String string = "{\"userName\":\"zhao\", \"curAge\":\"16\"}";
        User user2 = objectMapper.readValue(string, User.class);
        System.out.println(user2); // User(userName=null, curAge=16)
```

## 3.处理泛型

使用`TypeReference<ResultDTO<User>>`：

```java
ObjectMapper objectMapper = new ObjectMapper();

User user = new User("zhao", 16);
ResultDTO<User> resultDTO = new ResultDTO.Builder<User>(user)
    .code("success")
    .message("test")
    .build();
String writeValue = objectMapper.writeValueAsString(resultDTO);
// 带泛型的反序列化
ResultDTO<User> readValue = objectMapper.readValue(writeValue, new TypeReference<ResultDTO<User>>() {});
System.out.println(readValue);
```

> **补充**
>
> ```java
> ObjectReader reader = Wrapper.getObjectMapper().readerFor(TypeFactory.defaultInstance().constructCollectionType(List.class, clazz));
> result = reader.readValue(Wrapper.writeValueAsString(response.getBody().getResponse()));
> // or
> new ObjectMapper().readValue(jsonFile, CollectionsTypeFactory.listOf(TypeFactory.defaultInstance().constructCollectionType(List.class, clazz)));
> ```

自定义`ResultDTO`类

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
class ResultDTO<T> {

    private String code;
    private String message;
    private T t;

    private ResultDTO(Builder<T> builder) {
        code = builder.code;
        message = builder.message;
        t = builder.t;
    }

    public static final class Builder<T> {
        private String code;
        private String message;
        private T t;

        public Builder(T t) {
            this.t = t;
        }

        public Builder<T> code(String code) {
            this.code = code;
            return this;
        }

        public Builder<T> message(String message) {
            this.message = message;
            return this;
        }

        public ResultDTO<T> build() {
            return new ResultDTO<>(this);
        }
    }
}
```

## 4.处理日期

### 4.1 Date类型

```java
ObjectMapper objectMapper = new ObjectMapper();
objectMapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
User user = new User();
user.setAge(123);
user.setDate(new Date(2022,10,4));
System.out.println(objectMapper.writeValueAsString(user)); // {"userName":null,"age":123,"date":"3922-11-04 00:00:00"}
```

> Date 不再推荐使用， 其打印出来的日期为`year + 1900，month + 1，date`。

### 4.2 Java8 API

默认不支持 Java8 的 API，例如`LocalDateTime`

实体类

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public class User {
    private String name;
    private Integer age;
    private LocalDateTime localDateTime;
}
```

序列化

```java
ObjectMapper objectMapper = new ObjectMapper();
User zhao = new User();
zhao.setName("zhao");
zhao.setLocalDateTime(LocalDateTime.of(2022, 6, 30, 16, 54, 0));
String string = objectMapper.writeValueAsString(zhao);
System.out.println(string);
```

发生异常

```bash
Exception in thread "main" com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Java 8 date/time type `java.time.LocalDateTime` not supported by default: add Module "com.fasterxml.jackson.datatype:jackson-datatype-jsr310" to enable handling (through reference chain: User["localDateTime"])
```

### 4.3 配置JavaTimeModule

导入依赖

```xml
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
    <version>2.13.3</version>
</dependency>
```

配置`objectMapper`

```java
ObjectMapper objectMapper = new ObjectMapper();
// 设置自动发现注册 TimeModule
objectMapper.findAndRegisterModules();
User zhao = new User();
zhao.setName("zhao");
zhao.setLocalDateTime(LocalDateTime.of(2022, 6, 30, 16, 54, 0));
String string = objectMapper.writeValueAsString(zhao);
System.out.println(string);
```

但此时输出格式并不理想：

```json
{"name":"zhao","localDateTime":[2022,6,30,16,54]}
```

#### 1.全局设置输出格式

```java
ObjectMapper objectMapper = new ObjectMapper();

// 配置 javaTimeModule
JavaTimeModule javaTimeModule = new JavaTimeModule();
javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(DateTimeFormatter.ofPattern("yyyy-MM-dd HH.mm.ss")));
javaTimeModule.addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern("yyyy-MM-dd HH.mm.ss")));
// 添加到 objectMapper 中
objectMapper.registerModule(javaTimeModule);

User zhao = new User();
zhao.setName("zhao");
zhao.setLocalDateTime(LocalDateTime.of(2022, 6, 30, 16, 54, 0));
String string = objectMapper.writeValueAsString(zhao);
System.out.println(string);
```

#### 2.`@JsonFormat`

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public class User {
    private String name;
    private Integer age;
    @JsonFormat(pattern="yyyy-MM-dd HH.mm.ss", timezone = "GMT+8")
    private LocalDateTime localDateTime;
}
```

```java
ObjectMapper objectMapper = new ObjectMapper();
// 自动注册 Module
objectMapper.findAndRegisterModules();
User zhao = new User();
zhao.setName("zhao");
zhao.setLocalDateTime(LocalDateTime.of(2022, 6, 30, 16, 54, 0));
String string = objectMapper.writeValueAsString(zhao);
System.out.println(string);
```

> 该注解同样适用于`Date`类型，但不推荐。

## 5.更新对象

对象的合并、重写，如果后者的属性有值，则用后者，否则用前者：

```java
ObjectMapper objectMapper = new ObjectMapper();

User user1 = new User("zhao", 15);
User user2 = new User();
user2.setCurAge(13);

User updatedUser = objectMapper.updateValue(user1, user2);
System.out.println(updatedUser); // User(userName=zhao, curAge=13)
```

## 6.转换byte数组

```java
ObjectMapper objectMapper = new ObjectMapper();
User user1 = new User("zhao", 16);
// 对象转为 byte 数组
byte[] byteArr = objectMapper.writeValueAsBytes(user1);
System.out.println(Arrays.toString(byteArr));
// byte 数组转为对象
User user2 = objectMapper.readValue(byteArr, User.class);
System.out.println(user2);
```

## 7.JsonParser

```java
JSONParser jsonParser = new JSONParser();
return jsonParser.parse(new InputStreamReader(new CLassPathResource(filename).getInputStream()));
```

## 8.SpringBoot整合

SpringBoot 中提供了构造 ObjectMapper 的类 `Jackson2ObjectMapperBuilder`。

```java
public ObjectMapper objectMapper() {
    Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder();
  
    // 通过 featuresToDisable() 关闭某些 feature
    builder.featuresToDisable(
    	SerializationFeature.WRITE_DATES_AS_TIMESTAMPS
    );
    
    // 通过 featuresToEnable() 开启某些 feature
    builder.featuresToEnable(
    	SerializationFeature.WRITE_DATES_WITH_ZONE_ID,
        DeserializationFeature.ACCEPE_EMPTY_STRING_AS_NULL_OBJECT
    );
    
    // 自带方法
    builder.indentOutput(true);
    builder.failOnEmptyBeans(false);
    builder.failOnUnknowProperties(false);
    
    // 通过 serializationInclusion() 设置序列化的规则
    builder.serializationInclusion(JsonInclude.Include.NON_NULL);
    
    // 添加模式
    builder.modules(new GeoJsonModule(), new JavaTimeModule(), new MoneyModule());
    
    return builder.build(); // 返回 ObjectMapper
}
```

**最佳实践**

```java
@Bean
public ObjectMapper objectMapper() {
    Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder();
    builder.modules(new JavaTimeModule())
            .featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, SerializationFeature.WRITE_DATES_WITH_CONTEXT_TIME_ZONE, 
                    SerializationFeature.FAIL_ON_EMPTY_BEANS, DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, DeserializationFeature.ADJUST_DATES_TO_CONTEXT_TIME_ZONE)
            .featuresToEnable(DeserializationFeature.ACCEPT_EMPTY_STRING_AS_NULL_OBJECT)
            .serializationInclusion(JsonInclude.Include.NON_NULL);
    return builder.build();
}
```

