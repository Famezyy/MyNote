## 示例需求及构造数据

```java
@Getter
@Setter
@ToString
public class Person {
    /**人员ID*/
    private Integer id;
    private String name;
    private Integer age;
     /**销售额*/
    private BigDecimal sales;
}

@Getter
@Setter
@ToString
public class Sum {
    /**平均年龄*/
    private Double avgAge;
    /**总销售额*/
    private BigDecimal sumSale;
}
```

创建几个测试数据

```java
public void inintPersons() {
    // 人员列表
    String idKeys = "mylua:id";
    // 销售人员详情
    String userkey = "mylua:user";
    List<Object> ids = ImmutableList.of(1, 2, 3, 4);
    ids.forEach(p -> {
        Integer i = (Integer) p;
        Person person = new Person();
        person.setId(i);
        person.setName("user" + i);
        person.setAge(30 + i);
        person.setSales(new BigDecimal(i * 30000));

        redisUtils.hset(userkey, String.valueOf(p), person);
    });

    redisUtils.lsetlist(idKeys, ids);
}
```

初始化脚本后，Redis中将包含一组测试数据：

> key=mylua:id 用户ID列表，目前采用列表实现。可以使用有序集合实现，按用户销售额进行排序，实现销售人员列表展示
>
> 
>
> key=mylua:user 散列表实现，item=用户ID，存储用户详情，{id: "", name: "", age: 1, sales: 1}

## 执行Lua脚本

- 读取销售人员ID列表
- 循环列表读取每个销售信息，累加年龄及销售额
- 计算年龄平均值，返回人均年龄和总销售

在实现这个脚本时，只需要传入两个键值，销售人员ID列表键，散列表得键，用户ID由中间过程产生；不需要传递参数。因此其脚本参见下述代码

```lua
-- 调用方式：2 idlistKey userKey, 返回{avgAge, sumSale}
-- 读取参数键，分别为id集合key，用户详情key
local idsKey = KEYS[1]
local userKey = KEYS[2]
-- 记录日志，安装默认的log级别为NOTICE
redis.log(redis.LOG_NOTICE, "Receie the param:" .. idsKey)

-- 获取所有的id集合, table
local ids = redis.call('lrange', idsKey, 0, -1)
-- 记录用户长度
redis.log(redis.LOG_NOTICE, "the user count:" .. #ids)

local sumAge = 0

-- 结果 销售总额，人均年龄
local sumSale = 0
local avgAge = 0

if type(ids) == "table" and #ids > 0 then
    for _, v in pairs(ids) do
        -- 散列表中存储的是字符串，因为序列化为JSON，因此使用cjson将json字符串转为Lua table
        local user = cjson.decode(redis.call("hget", userKey, v))
        sumAge = sumAge + user.age
        sumSale = sumSale + user.sales
    end
    avgAge = sumAge / #ids
end

local result = {}
result.avgAge = avgAge
result.sumSale = sumSale

-- table返回时转换为JSON
local ret = cjson.encode(result)
redis.log(redis.LOG_NOTICE, "calc result:" .. ret)

return ret
```

> 上面的脚本中使用到了 CJSON 类库，该类库由 C 开发，在 LUA 中提供高效的 JSON 操作，常用方法有
>
> - cjson.encode({}) 将 Lua 的 table 数据类型转换为 JSON 格式字符串
> - cjson.decode(jsonString) 将 json 格式字符串转为 Lua 的 table 数据

脚本完成之后，添加到 SpringBoot 项目的 resources 目录下，即目录结构为：src/resources/person.lua

RedisTemplate提供两种方法可以执行Lua脚本：

```java
/**
    script为脚本资源，keys为一个数组形式的Key集合，最后为可选参数
*/
<T> T execute(RedisScript<T> script, List<K> keys, Object... args)

/**
    和上述中方式一致，多了两个参数，分别为参数序列化方法、结果序列化方法，更方便扩展
*/
<T> T execute(RedisScript<T> script, RedisSerializer<?> argsSerializer, RedisSerializer<T> resultSerializer, List<K> keys, Object... args)
```

​    RedisTemplate 使用 Lua 脚本时，有一个非常重要的参数 RedisScript，该参数一方面决定了Lua脚本资源，另一方面也决定了返回值类型。在实际开发过程中，使用的是默认 DefaultRedisScript 实现对象，对这个类进行一些查阅，了解以下使用方式：

```java
// 这里代码不全，可以查看org.springframework.data.redis.core.script.DefaultRedisScript
// 三个属性，分别表示了脚本资源，脚本sha1，以及结果类型
private @Nullable ScriptSource scriptSource;
private @Nullable String sha1;
private @Nullable Class<T> resultType;
// 脚本字符串和返回类型构造函数,一般不常用
public DefaultRedisScript(String script, @Nullable Class<T> resultType) {
        ...
}
// 获取脚本Sha1，缓存时使用
public String getSha1() {
    ...
}
// 设置返回类型
// The script result type. Should be one of Long, Boolean, List, or deserialized value type
public void setResultType(Class<T> resultType) {
    ...
}
// 设置脚本资源
public void setLocation(Resource scriptLocation) {
    this.scriptSource = new ResourceScriptSource(scriptLocation);
}
// 设置脚本资源
public void setScriptSource(ScriptSource scriptSource) {
    this.scriptSource = scriptSource;
}
```

> 上面setResultType方法中，有一个要求，只能设置类型为：Long、布尔、List或者反序列化类型，设置为什么类型，最后就得到什么类型。

​    在 SpringBoot 项目中，脚本一般放在 resources 中，因此很容易获取到 Resource 资源（ClassPathResource），因此后面两个设置脚本资源的方法是比较方便的。对于两个执行 Lua 脚本的方法，分别进行调用，看一下实现的方式

- **直接调用，不提供序列化方法**

  ```java
  T execute(RedisScript script, List keys, Object... args)
  ```

  ```java
  public void execLuaWithoutSerializer() {
      DefaultRedisScript<JSONObject> script = new DefaultRedisScript<>();
      script.setResultType(JSONObject.class);
      script.setLocation(new ClassPathResource("/person.lua"));
      List<String> keys = ImmutableList.of("mylua:id", "mylua:user");
      JSONObject sum = redisTemplate.execute(script, keys);
      System.out.println(sum.toJSONString());
  }
  ```

  ​    在之前的脚本中，定义了必须传递两个 key，但不需要参数，因此在调用时传入了一个 List，存储两个 key。由于脚本之后返回了一个 JSON 格式字符串，并且 RedisTemplate 采用了 FastJson 序化，因此返回一个 JSONObject，**调用时设置的ResultType必须和脚本返回类型一致**，由于没有指定序列化，会使用默认的序列化工具，而设置的默认序列化方法泛型为Object，因此上述无法直接使用 Sum 类型，参见第一节 RedisTemplate redisTemplate。如果有序列化特定类型，还是需要采用更明确第二种方法。执行后，其返回的结果如下

  ```json
  {"sumSale":300000,"avgAge":32.5}
  ```

  ​    由于团队采用了 RedisTemplate 的泛型方案，虽然能够处理任意类型，但是对于类型转换确实存在一些不方便之处，本例中只能只能转换为 JSONObject 也是基于此，无法直接序列化为 Sum对象，只能再次提供一个新的序列化再次进行。可以采用 RedisTemplate 不指定泛型的方式去解决类型转换的问题，但使用起来也会有一些不变之处。

- **提供序列化方法**

  ```java
  T execute(RedisScript script, RedisSerializer argsSerializer, RedisSerializer resultSerializer, List keys, Object... args)
  ```

  ```java
  public void execLuaWithSerializer() {
      DefaultRedisScript<Sum> script = new DefaultRedisScript<>();
      script.setResultType(Sum.class);
      script.setLocation(new ClassPathResource("/person.lua"));
      List<String> keys = ImmutableList.of("mylua:id", "mylua:user");
      // FastJsonRedisSerializer 为自定义的序列化类
      Sum sum = redisTemplate.execute(script, new StringRedisSerializer(), new FastJsonRedisSerializer<>(Sum.class), keys);
      System.out.println(sum);
  }
  ```

  在这个示例中，期望返回自定义的Sum类型，将更方便在程序中的应用。主要就是返回结果的序列化方法，也使用了同一个序列化操作对象。其执行结果

  ```java
  Sum(avgAge=32.5, sumSale=300000)
  ```

  ​    执行结果符合预期。上面两种方法都可以使用，相对来说，第二种更适合在业务开发中使用。由于业务的不同，因此使用的脚本不同，返回数据不同，键与参数都不同。在使用过程中可以稍微封装一下。

## SpringBoot执行脚本流程分析

​    在之前介绍执行脚本指令时，提到过两种指令EVAL、EVALSHA，并提供了检查脚本缓存是否存在的指令SCRIPT EXISTS，EVAL指令或者SCRIPT LOAD指令都可以将脚本缓存，EVAL立即执行，但不会返回SHA1，SCRIPT LOAD缓存脚本，但不立即执行，并且返回SHA1值。EVALSHA指令可以直接利用Redis缓存的脚本执行，而不需要每次都传递脚本，当脚本比较大时，可以节约网络传输数据量。

​    但是在上面SpringBoot执行过程中，并没有发现其调用EVALSHA，也没有执行SCRIPT EXISTS的方法，这个过程中有没有利用到SHA(*在RedisScript中，有一个getSha1的方法*)，需要分析一下其执行流程。

上面的两种方法最终都调用到下述方法

```java
// 实现类org.springframework.data.redis.core.script.DefaultScriptExecutor
// 第一个方法调用该方法，采用过了 RedisTemplate 默认提供的序列化对象
public <T> T execute(final RedisScript<T> script, final List<K> keys, final Object... args) {
    // use the Template's value serializer for args and result
    return execute(script, template.getValueSerializer(), (RedisSerializer<T>) template.getValueSerializer(), keys, args);
}

// 最终两个都调用了该方法
public <T> T execute(final RedisScript<T> script, final RedisSerializer<?> argsSerializer,
                     final RedisSerializer<T> resultSerializer, final List<K> keys, final Object... args) {
    return template.execute((RedisCallback<T>) connection -> {
        final ReturnType returnType = ReturnType.fromJavaType(script.getResultType());
        final byte[][] keysAndArgs = keysAndArgs(argsSerializer, keys, args);
        final int keySize = keys != null ? keys.size() : 0;
        if (connection.isPipelined() || connection.isQueueing()) {
            connection.eval(scriptBytes(script), returnType, keySize, keysAndArgs);
            return null;
        }
        // 没有使用管道，调用该方法
        return eval(connection, script, returnType, keySize, keysAndArgs, resultSerializer);
    });
}

protected <T> T eval(RedisConnection connection, RedisScript<T> script, ReturnType returnType, int numKeys,
                     byte[][] keysAndArgs, RedisSerializer<T> resultSerializer) {

    Object result;
    try {
        // 先尝试执行 redis 中缓存的 lua 脚本
        result = connection.evalSha(script.getSha1(), returnType, numKeys, keysAndArgs);
    } catch (Exception e) {

        if (!ScriptUtils.exceptionContainsNoScriptError(e)) {
            throw e instanceof RuntimeException ? (RuntimeException) e : new RedisSystemException(e.getMessage(), e);
        }

        result = connection.eval(scriptBytes(script), returnType, numKeys, keysAndArgs);
    }

    if (script.getResultType() == null) {
        return null;
    }

    return deserializeResult(resultSerializer, result);
}
```

​    从上面最后一个方法执行中可以看到，SpringBoot 先计算了当前资源的 Sha1，并使用 EVALSHA 指令尝试执行了一次，如果成功，则返回结果，如果缓存没有该脚本，则进入异常部分，并最终使用了EVAL指令进行执行。

​    从这里可以看到，SpringBoot 客户端已经实现了脚本缓存的功能，只不过进行了封装，并且不对用户暴露。在使用时简单，傻瓜，并且用起来很舒服。总结一下就是：SpringBoot 每次都先按EVALSHA 执行，没有缓存脚本，再次执行EVAL，得到结果并缓存脚本。

> ==注：==开发环境是 Windows，Redis 在 Linux 上部署，由于编码以及文件的换行符配置导致 Windows 下计算的 SHA1，与 Redis 在 Linux 下缓存的文件 SHA1 不匹配，导致每次都无法命中缓存，此时可以通过 IDEA 的文件换行设置，调整脚本文件使用 Unix 换行符，可以解决不同系统匹配问题。
>
> <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/9095350-5186dc7073f4f01d-461403208c6dcd04ba28db87156ad896-f5087b.png" alt="img" style="zoom:80%;" />
>
> IDEA设置Linux换行符
>
> *如上图，最开始默认为Windows换行符，CRLF，调整为LF即可解决上述问题。*