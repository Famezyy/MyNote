# 第03章_Nginx Lua

Nginx 还具备可编程能力，理论上可以使用 Nginx 的扩展组件 ngx_lua 开发各种复杂的动态应用。不过由于 Lua 是一种脚本动态语言，因此不太适合做复杂业务逻辑的程序开发。但是在高并发场景下，nginx Lua 编程是解决性能问题的利器。

## 1.简介

### 1.1 应用场景

Nginx Lua 编程的主要应用场景如下：

**（1）API 网管**

实现数据校验前置、请求过滤、API 请求聚合、AB 测试、灰度发布、降级、监控等功能，开源网关 Kong 就是基于 Nginx Lua 开发的。

**（2）高速缓存**

可以对响应内容进行缓存、减少到后端的请求、从而提升性能。比如 Nginx Lua 可以和 Java 容器（如 Tomcat）、Redis 整合，由 Java 容器进行业务处理和数据缓存，而 Nginx 负责读缓存并进行响应。

**（3）简单的动态 Web 应用**

可以完成一些业务逻辑处理较少但是耗费 CPU 的简单应用，比如模板页面的渲染。一般的 Nginx Lua 页面渲染处理流程为：从 Redis 获取业务处理结果数据，从本地加载 XML/HTML 页面模板，然后进行页面渲染。

**（4）网关限流**

缓存、降级、限流是解决高并发的三大利器，Nginx 内置了令牌限流的算法，但是对于分布式的限流场景，可以通过 Nginx Lua 编程定制自己的限流机制。

### 1.2 ngx_lua简介

Lua 是一种轻量级、可嵌入式的脚本语言，可以非常容易地嵌入其他语言中使用。例如在 Nginx 中嵌入 Lua VM（Lua 虚拟机），请求时创建一个 VM，请求结束时回收 VM。

`ngx_lua` 是 Nginx 的一个扩展模块，将 Lua VM 嵌入 Nginx 中，从而可以在 Nginx 内部运行 Lua 脚本。`ngx_lua` 提供了与 Nginx 交互的许多 API，它和 `Servlet` 类似。使用 `ngx_lua` 开发 Web 应用时，有很多源码的 Lua 基础性模块可供使用，比如 OpenResty 就提供了一些常用的 `ngx_lua` 开发模块：

- `lua-resty-memcached`：操作 Memcached 缓存
- `lua-resty-mysql`：操作 MySQL 数据库
- `lua-resty-redis`：操作 Redis 缓存
- `lua-resty-dns`：操作 DNS 域名服务器
- `lua-resty-limit-traffic`：限流
- `lua-resty-template`：进行模版的渲染

除此之外还有许多其他第三方的 `ngx_lua` 插件，如 `lua-resty-jwt`、`lua-resty-kafka` 等，还可以自己开发自己的 Lua 模块。

## 2.Lua开发基础

Lua 脚本需要通过 Lua 解释器来解释执行，除了 Lua 官方的默认解释器外，目前使用广泛的 Lua 解释器叫做 LuaJIT。LuaJIT 是采用 C 语言编写的 Lua 脚本解释器。LuaJIT 被设计成全兼容标准 Lua 5.1，因此 LuaJIT 代码的语法和标准 Lua 的语法没多大区别，但是 LuaJIT 的运行速度比标准 Lua 快数十倍。

### 2.1 Lua模块的定义

与 Java 类似，实际开发的 Lua 代码需要进行分模块开发，Lua 中的一个模块对应一个 Lua 脚本文件。使用 `require` 指令导入 Lua 模块，第一次导入模块后，所有 Nginx 进程全局共享模块的数据和代码，每个 Worker 进程需要时会得到此模块的一个副本，不需要重复导入，从而提高 Lua 应用的性能。

下面演示一个简单的 Lua 模块，用来存放共有的基础对象和基础函数。

```lua
-- nginx/luaScript/module/common/basic.lua

-- 定义一个应用程序共有的 Lua 对象 app_info
local app_info = {
    version = "0.10"
}
-- 增加一个 path 属性，保存 Nginx 进程所保存的 Lua 模块路径，包括 conf 文件配置的部分路径
app_info.path = package.path;

-- 局部函数，获得最大值
local function max(num1, num2)
    if (num1 > num2) then
        result = num1;
    else
        result = num2;
    end
    return result;
end

-- 统一的模块对象
local _Module = {
    app_info = app_info;
    max = max;
}

return _Module
```

模块内的所有对象、数据、函数都定义成局部变量或者局部函数。然后对于需要暴露给外部的对象或者函数，作为成员属性保存到一个统一的 Lua 局部对象（如 `_Module`）中，通过返回这个统一的局部对象将内部的成员对象或者方法暴露出去，从而实现 Lua 的模块化封装。

### 2.2 Lua模块的使用

接下来创建一个 Lua 脚本来调用前面定义的这个基础模块。

```lua
-- nginx/luaScript/module/demo/helloworld.lua

-- 导入自定义的模块
local basic = require("luaScript.module.common.basic");

-- 使用模块的成员属性
ngx.say("Lua path is: " .. basic.app_info.path);
ngx.say("<br>");
-- 使用模块的成员方法
ngx.say("max 1 and 11 is: " .. basic.max(1,11));
```

在使用 `require` 内置函数导入 Lua 模块时，对于多级目录下的模块，使用 `require("目录1.目录2.模块名")` 的形式进行加载，源目录之间的 `"/"` 斜杠分隔符改成 `"."` 点号分隔符。

Lua 文件查找时，首先会在 Nginx 的当前工作目录查找，如果没有找到，就会在 Nginx 的 Lua 包路径 `lua_package_path` 和 `lua_package_cpath` 声明的位置查找。这两个参数需要在 nginx.conf 配置文件中进行配置：

```nginx
lua_package_path "/temp/lualibs/?/?.lua;;";
lua_package_cpath "/temp/lualibs/?.so;;";
```

`lua_package_path` 用于配置 Lua 文件的包路径；`lua_package_cpath` 用于配置 C 语言模块文件的包路径。在 Linux 系统上 C 语言模块文件的类型是 `".so"`；在 Windows 上是 `".dll"`。

Lua 包路径如果需要配置多个路径，那么路径之间是用分号 `";"` 分隔。末尾的两个分号 `";;"` 表示加上 Nginx 默认的 Lua 包搜索路径，其中包含 Nginx 的安装目录下的 lua 目录。

```bash
# 默认的 Lua 文件搜索路径
/usr/local/openresty/site/lualib/?.ljbc;
/usr/local/openresty/site/lualib/?/init.ljbc;
/usr/local/openresty/lualib/?.ljbc;
/usr/local/openresty/lualib/?/init.ljbc;
/usr/local/openresty/site/lualib/?.lua;
/usr/local/openresty/site/lualib/?/init.lua;
/usr/local/openresty/lualib/?.lua;
/usr/local/openresty/lualib/?/init.lua;
./?.lua;
/usr/local/openresty/luajit/share/luajit-2.1.0-beta3/?.lua;
/usr/local/share/lua/5.1/?.lua;
/usr/local/share/lua/5.1/?/init.lua;
/usr/local/openresty/luajit/share/lua/5.1/?.lua;
/usr/local/openresty/luajit/share/lua/5.1/?/init.lua
```

在 OpenResty 的 lualib 下已经提供了大量第三方开发库，如 CJSON、Redis 客户端、MySQL 客户端等，并且这些 Lua 模块已经包含到默认的搜索路径中。OpenResty 的 lualib 下的模块可以直接在 Lua 文件中通过 `require` 方式导入：

```lua
local redis = require("resty.redis");
local cjson = require("cjson");
```

在 nginx.conf 配置文件中使用 `content_by_lua_file` 来调用定义好的模块：

```nginx
location /helloworld {
    default_type 'text/html';
    charset utf-8;
    content_by_lua_file luaScript/module/demo/helloworld.lua;
}
```

在 `/nginx` 目录下执行：

```bash
nginx -p . -c conf/my-nginx.conf -s reload
```

访问 `/helloworld` 得到以下结果：

```bash
Lua path is: /usr/local/openresty/site/lualib/?.ljbc;/usr/local/openresty/site/lualib/?/init.ljbc;/usr/local/openresty/lualib/?.ljbc;/usr/local/openresty/lualib/?/init.ljbc;/usr/local/openresty/site/lualib/?.lua;/usr/local/openresty/site/lualib/?/init.lua;/usr/local/openresty/lualib/?.lua;/usr/local/openresty/lualib/?/init.lua;./?.lua;/usr/local/openresty/luajit/share/luajit-2.1.0-beta3/?.lua;/usr/local/share/lua/5.1/?.lua;/usr/local/share/lua/5.1/?/init.lua;/usr/local/openresty/luajit/share/lua/5.1/?.lua;/usr/local/openresty/luajit/share/lua/5.1/?/init.lua
max 1 and 11 is: 11
```

### 2.3 Lua的数据类型

#### 1.八种数据类型

|    类型    |   名称   | 说明                                                         |
| :--------: | :------: | ------------------------------------------------------------ |
|  `number`  |   实数   | 可以是整数、浮点数                                           |
|  `string`  |  字符串  | 字符串类型，值是不可改变的                                   |
| `boolean`  | 布尔类型 | false 和 `nil` 是假，其他为真                                |
|  `table`   | 数组容器 | `table` 类型实现了一种抽象的“关联数组”，相当于 Java 中的 `Map` |
| `userdata` |    类    | 其他语言中的对象类型，转换过来就变成 `userdata` 类型，比如 Redis 返回的空值有可能就是 `userdata` 类型，判空的时候要注意！ |
|  `thread`  |   线程   | 和 Java 中的线程差不多，代表一条执行序列，拥有自己独立的栈、局部变量和命令指针 |
| `function` |   函数   | 由 C 或 Lua 编写的函数，属于一种数据类型                     |
|   `nil`    |  空类型  | 表示变量没被赋值，`nil` 类型就 `nil` 一个值，变量赋值成 `nil` 也表示删除变量 |

Lua 是弱类型语言，和 JavaScript 等脚本语言类似，变量没有固定的数据类型，每个变量可以包含任意类型的值。使用内置的 `type()` 方法可以获取该变量的数据类型。

```lua
-- 在 basic 模块中添加以下函数

-- 在屏幕上打印日志
local function log_screen(...)
    -- 这里的 ... 和 {} 符号中间需要有空格号，否则会出错
    local args = { ... };
    for i, v in pairs(args) do
        -- print 是打印到标准输出，同 Java 的 System.out.print
        -- print("index:", i, " value:", v);
        -- ".." 是字符串连接操作符
        ngx.say(tostring(v) .. ",");
    end
    ngx.say("<br>");
end

local _Module = {
    log_screen = log_screen;
}
```

定义一个输出数据类型的模块

```lua
-- nginx/luaScript/module/demo/dataType.lua

local basic = require("luaScript/module/common/basic")

local function showDataType()
    local i;
    basic.log_screen("字符串的类型", type("hello world"));
    basic.log_screen("方法的类型", type(showDataType));
    basic.log_screen("true 的类型", type(true));
    basic.log_screen("整数数字的类型", type(360));
    basic.log_screen("浮点数字的类型", type(360.0));
    basic.log_screen("nil 的类型", type(nil));
    basic.log_screen("未赋值变量 i 的类型", type(i));
end

local _Module = {
    showDataType = showDataType;
}

return _Module
```

再定义一个调用该模块的模块

```lua
-- nginx/luaScript/module/demo/runDemo.lua

local dataType = require("luaScript.module.demo.dataType");
ngx.say("下面是数据类型的输出结果：<br>");
dataType.showDataType();
```

在 Nginx 的配置文件中调用 runDemo 后通过浏览器查看结果：

```bash
下面是数据类型的输出结果：
字符串的类型, string,
方法的类型, function,
true 的类型, boolean,
整数数字的类型, number,
浮点数字的类型, number,
nil 的类型, nil,
未赋值变量 i 的类型, nil,
```

#### 2.注意事项

- `nil` 是一种类型，也是一个值，如果变量没有被赋值，那么类型和值都是 `nil`；与 Nginx 不同的是，OpenResty 还提供了一种特殊的空值，即 `ngx.null`，用来表示空值，但是不同于 `nil`。

- `boolean` 的可选值为 true 和 false。在 Lua 中，只有 `nil` 和 false 为“假”，其他所有值均为“真“，比如数字 0 和空字符串都是”真“。

- `number` 类型用于表示实数，与 Java 的 `double` 类似，但是又有区别，Lua 的整数类型也是 `number`。一般来说，Lua 中的 `number` 类型是用双精度浮点数来实现的。可以使用数学函数 `math.lua` 来操作 `number` 类型的变量。在 `math.lua` 模块中定义了大量数字操作方法，比如定义了 `floor` 和 `ceil` 等操作。

  ```lua
  -- nginx/luaScript/module/demo/dataType.lua
  
  -- 演示取整操作
  local function intPart(number)
      basic.log_screen("演示的整数", number);
      basic.log_screen("向下取整", math.floor(number));
      basic.log_screen("向上取整", math.ceil(number));
  end
  ```

  在 runDemo 中调用该方法：

  ```lua
  -- nginx/luaScript/module/demo/runDemo.lua
  
  ngx.say("下面是数字取整的输出结果：<br>");
  dataType.intPart(0.01);
  dataType.intPart(3.14);
  ```

  结果如下：

  ```bash
  下面是数字取整的输出结果：
  演示的整数, 0.01,
  向下取整, 0,
  向上取整, 1,
  演示的整数, 3.14,
  向下取整, 3,
  向上取整, 4,
  ```

- `table` 类型实现了一种抽象的“关联数组”，相当于 Java 中的 `Map`。`table` 的索引通常是 `number` 类型或者 `string` 类型，也可以是除 `nil` 以外的任意类型的值。默认情况下，`table` 中的 key 是 `number` 类型的，并且 key 的值为递增的数字。

- Lua 中的函数也是一种类型 `function`。函数可以存储在变量中，可以作为参数传递给其他函数，还可以作为其他函数的返回值。定义一个有名字的函数本质上是定义一个函数对象，然后赋值给变量名称。

  上面的 `max` 函数可以改写为以下形式：

  ```java
  local max = function (num1, num2)
      ...
  end
  ```

### 2.4 Lua的字符串

Lua 中有 3 种方式表示字符串：

- 使用一对匹配的半角英文单引号，例如 `'hello'`
- 使用一对匹配的半角英文双引号，例如 `"hello"`
- 使用一种双方括号 `[[]]` 括起来的方式定义，例如 `[["add\name", 'hello']]`。双方括号内的任何转义字符不会被处理，例如 `\n` 就不会被转义

和 Java 一样，Lua 的字符串的值是不可改变的，如果需要改变，就需要根据修改要求来创建一个新的字符串并返回。另外 Lua **==不支持通过下标来访问==**字符串的某个字符。

Lua 定义了一个负责字符串操作的 `string` 模块，包含很多强大的字符串操作函数。主要的字符串操作介绍如下：

- `..`：字符串拼接符号

  ```lua
  local here = "hello " .. "world";
  ```

- `string.len(s)`：获取字符串的长度

  接受一个字符串作为参数，返回它的长度。功能和 `#` 运算符类似，后者也是获取字符串的长度。实际开发中建议使用 `#` 运算符获取字符串长度。

  ```lua
  -- 可以直接使用 string 的函数
  ngx.say(string.len(here));
  ngx.say("<br>");
  ngx.say(#here);
  ```

- `string.format(formatString, ...)`：格式化字符串

  第一个参数 `formatString` 表示需要进行格式化的字符串规则，通常由常规文本和格式指令组成，比如：

  ```lua
  string.format("保留四位小数的圆周率 %.4f", 3.1415926); -- 保留四位小数的圆周率：3.1416
  string.format("%s %02d-%02d-%02d", "今天是：", 2020, 1, 1); -- 今天是： 2020-01-01
  ```

  在 `formatString` 参数中，除了常规文本之外，还有格式指令。格式指令由 `%` 加上一个类型字母组成，比如 `%s`（字符串格式化）、`%d`（整数格式化）、`%f`（浮点数格式化）等，在 `%` 和类型符号的中间可以选择性地加上一些格式控制数据，比如 `%02d`，表示进行两位的整数格式输出。

  `format` 函数后面的参数是一个可变长参数，表示一系列需要进行格式化的值。一般来说，前面的 `formatString` 参数中有多少格式化指令，后面就需要放置对应数量的参数值，并且后面的参数类型需要与 `formatString` 参数中对应位置的格式化指令中的类型符号相匹配。如果用 `%d` 匹配数字以外的字符则会出错。

- `string.find(s, pattern[, init[, plain]])`：字符串匹配

  在 s 字符串中查找第一个匹配正则表达式 `pattern` 的子字符串，返回第一次在 s 中出现的满足条件的子串开始位置和结束位置。如果匹配失败则返回 `nil`。第三个参数 `init` 默认为 1，表示从起始位置 1 开始找。第 4 个参数值默认为 false，表示第二个参数 `pattern` 为正则表达式，如果设置为 true 则只会把 `pattern` 看成一个普通字符串。

  ```lua
  s = "abc \"it's a cat\"";
  -- [\"'] 表示匹配 " 或者 '
  result = {string.find(s, "([\"'])(.-)%1")};
  for i, v in pairs(result) do
      ngx.say("index: " .. i .. ";" .. "value: " .. v);
      ngx.say("<br>");
  end
  ```

  结果

  ```bash
  index: 1;value: 5
  index: 2;value: 16
  index: 3;value: "
  index: 4;value: it's a cat
  ```

  > **注意**
  >
  > Lua 里的正则匹配是缩水版的，仅支持以下模式：
  >
  > - `.`：任意字符
  > - `%s`：空白符
  > - `%p`：标点
  > - `%c`：控制字符
  > - `%d`：数字
  > - `%x`：十六进制数
  > - `%z`：代表 0 的字符
  > - `%a`：字母
  > - `%l`：小写字母
  > - `%u`：大写字母
  > - `%w`：字母数字
  > - `[]`：字符组，匹配任一字符
  > - `()`：捕获组
  >
  > 字符类的大写形式代表相应集合的补集， 比如 `%A` 表示除了字母以外的字符集。
  >
  > 另外，`*`、`+`、`-` 三个，作为通配符分别表示：
  >
  > - `*`：匹配前面指定的 0 或多个同类字符， 尽可能匹配更长的符合条件的字串
  > - `-`：匹配前面指定的 0 或多个同类字符， 尽可能匹配更短的符合条件的字串（非贪婪）
  > - `+`：匹配前面指定的 1 或多个同类字符， 尽可能匹配更长的符合条件的字串

- `string.upper(s)`、`string.lower(s)`：字符串大小写转换

  接收一个字符串 s，返回一个把所有小写字母变成大写字母的字符串。

  ```lua
  src = "Hello world!";
  string.upper(src); -- HELLO WORLD!
  string.lower(src); -- hello world!
  ```

### 2.5 Lua的数组容器

Lua 数组的类型定义和关键词为 `table`，和 Java 的数组对比起来有以下几个特点：

- 和 Java 的 HashMap 类似，Lua 数组内部实际采用哈希表保存键值对；不同的是，Lua 在初始化一个普通数组时，如果不显示地指定元素的 key，就会默认使用数字索引作为 key。

- 定义一个数组使用花括号，中间加上初始化的元素序列，元素之间以逗号隔开。

  ```lua
  local array1 = {"value1", "value2", "value3"};
  local array2 = {k1="value1", k2="value2", k3="value3"};
  ```

- 普通 Lua 数组的数字索引对应于 Java 的元素下标，但是 Lua 中==是从 1 开始计数的==。

- 普通 Lua 数组的长度计算从第一个元素开始，计算到最后一个非 `nil` 的元素为止，中间的元素数量就是长度。

- 取得数组元素值使用 `[]` 符号，形式为 `array[key]`，其中 `array` 代表数组变量名称，key 代表元素的索引；对于普通的数组，key 为元素的索引值；对于键值对类型的数组，key 就是键值对中的 key。

  ```lua
  for i = 1, 3 do
      ngx.say(i .. "=" .. array1[i] .. ","); -- 1=value1, 2=value2, 3=value3,
  end
  ngx.say("<br>");
  for k, v in pairs(array2) do
      ngx.say(k .. "=" .. array2[k] .. ","); -- k3=value3, k1=value1, k2=value2,
  end
  ```

数组的常用方法有如下几种：

此外 Lua 定义了一个负责数组和容器操作的 `table` 模块，主要的字符串操作大致如下：

- `table.getn(t)`：获取长度

  对于普通的数组，键从 1 到 n 的值不存在空值时，长度就为 n。如果有一个元素为空值，那么数组长度为空值前面部分的长度，空值后面不计算在内。

  和字符串相同，获取数组长度也可以使用 `#`，在 Lua 5.1 之后去掉了 `table.getn(t)`，此时可以直接使用 `#`。

  ```lua
  table.getn(array1); -- 3
  #array1;            -- 3
  ```

- `table.concat(array, [, sep, [, i, [, j]]])`：连接数组元素

  按照 `array[i] .. sep .. array[i + 1] .. sep .. array[j]` 的方式将普通数组中所有的元素连接成一个字符串并返回。分隔字符串 `sep` 默认为空白字符串。起始位置 `i` 默认为 1，结束位置 `j` 默认是 `array` 的长度。如果 `i` 大于 `j`，就返回一个空字符串。

  ```lua
  local testTab = {1, 2, 3, 4, 5, 6, 7};     -- 1234567
  ngx.say(table.concat(testTab));
  ngx.say(table.concat(testTab, "*", 1, 3)); -- 1*2*3
  ```

- `table.insert(array, [pos, ], value)`：插入元素

  在 `array` 的位置 `pos` 处插入元素 `value`，后面的元素向后顺移。`pos` 的默认值为 `#list + 1`，因此调用 `table.insert(array, x)` 会将 x 插在普通数组 `array` 的末尾。

  ```lua
  -- 调用方法前必须先声明
  local printTable = function (tab)
      ngx.say(table.concat(tab, ","));
      ngx.say("<br>");
  end
  
  local testTab = {1, 2, 3, 4};
  table.insert(testTab, 5);
  printTable(testTab);           -- 1,2,3,4,5
  table.insert(testTab, 2, 10);
  printTable(testTab);           -- 1,10,2,3,4,5
  ```

- `table.remove(array[, pos])`：删除元素

  删除 `array` 中 `pos` 位置上的元素，并返回这个被删除的值。当 `pos` 是 1 到 `#list` 之间的整数时，将后面的所有元素前移一位，并删除最后一个元素。

  ```lua
  testTab = {1, 2, 3, 4, 5, 6, 7};
  -- 删除最后一个元素
  table.remove(testTab);
  printTable(testTab);      -- 1,2,3,4,5,6
  -- 删除第 2 个元素
  table.remove(testTab, 2);
  printTable(testTab);      -- 1,3,4,5,6
  ```

### 2.6 Lua的控制结构

#### 1.if-else

**（1）单分支结构：if**

以关键词 `if` 开头，`end` 结尾。

```lua
local x = "demo"
if x == 'demo' then
    ngx.say("单分支结构");
end
```

**（2）两分支结构：if-else**

```lua
local x = "demo"
if x == 'demo2' then
    ngx.say('两分支结构')
else
    ngx.say('还是两分支结构') -- 还是两分支结构
end
```

**（3）多分支结构：if-elseif-else**

```lua
local x = "demo"
if x == 'demo1' then
    ngx.say("多分支结构")
elseif x == 'demo2' then
    ngx.say("依然多分支结构")
else
    ngx.say("还是多分支结构") -- 还是多分支结构
end
```

#### 2.for

**（1）基础 for 循环**

```lua
for var = begin, finish, step do
    -- body
end
```

基础 for 循环的语法中，`var` 表示迭代变量，`begin`、`finish`、`step` 表示控制的变量。迭代变量 `var` 从 `begin` 开始，一直到 `finish` 循环结束，每次变化都以 `step` 作为步长递增。`begin`、`finish`、`step` 可以是表达式，但是 3 个表达式只会在循环开始是执行一次。其中步长表达式 `step` 是可选的，如果没有设置默认是 1。迭代变量 `var` 的作用域仅在 `for` 循环内，在循环过程中不要改变迭代变量 `var` 的值。

 ```lua
 for int i = 1, 5, 2 do
     ngx.say(i .. " ") -- 1 3 5
 end
 ```

**（2）增强 foreach 循环**

```lua
for key, value in pairs(table) do
    -- body
end
```

在 Lua 的 `table` 内部保存有一个键值对的列表，`foreach` 循环就是对这个列表中的键值对进行迭代，`pairs(table)` 函数的作用就是取得 `table` 内部的键值对列表。

```lua
local days = {
    "Sunday", "Monday", "Tuesday", "Wednesday",
    "Thursday", "Friday", "Saturday"
}

for key, value in pairs(days) do
    ngx.say(key .. ":" .. value .. "; ");
end

ngx.say("<br>");

local days2 = {
    Sunday = 1, Monday = 2, Tuesday = 3, Wednesday = 4,
    Thursday = 5, Friday = 6, Saturday = 7
}

for key, value in pairs(days2) do
    ngx.say(key .. ":" .. value .. "; ");
end
```

结果

```bash
1:Sunday; 2:Monday; 3:Tuesday; 4:Wednesday; 5:Thursday; 6:Friday; 7:Saturday;
Wednesday:4; Thursday:5; Friday:6; Saturday:7; Sunday:1; Monday:2; Tuesday:3;
```

使用数字作为 key 时要加上 `"[]"`：

```lua
for i, v in pairs({[2]=test}) do
    print(i, v)
end
```

### 2.7 Lua的函数

#### 1.定义

Lua 函数使用关键词 `function` 来定义，在函数中使用局部变量，起作用范围不会超出函数。

```lua
optional_function_scope function function_name(arg1, arg2, ... argn)
    function_body
    return result_params_comma_separated
end
```

- `optional_function_scope`：表示所定义的函数是全局函数还是局部函数，该参数是可选参数，默认为全剧函数，如果定义为局部函数，那么设置关键字 `local`

  ```lua
  function testVar()
      globalVar = "I'm a global var";
      local localVar = "I'm a local var";
  end
  
  testVar();
  
  ngx.say(globalVar); -- I'm a global var
  ngx.say(localVar);  -- nil
  ```

- `function_name`：该参数用于指定函数名称

- `arg1, arg2, ... argn`：函数参数，多个参数以逗号隔开，也可以不带参数

- `function_body`：函数体，函数中需要执行的代码语句块

- `result_params_comma_separated`：函数返回值，Lua 语言中的函数可以返回多个值，每个值以逗号隔开

#### 2.可变参数

在函数参数列表中使用三点 `"..."` 表示函数有可变的参数。在函数的内部可以通过一个数组访问可变参数的实参列表。

```lua
local function log(...)
    local args = { ... }         -- "..." 和 "{}" 中间需要有空格，否则会出错
    for i, v in pairs(args) do
        ngx.say(v .. ",");
    end
    ngx.say("<br>")
end
```

#### 3.传参

Lua 中的函数的参数大部分是**值传递**。值传递就是调用函数时，把实参的值通过赋值传递给形参，然后形参的改变就不会影响实参了。但是 `table` 数字类型的传递方式是**引用传递**，传递的是实际参数的引用（内存地址）。

#### 4.返回值

Lua 允许函数返回多个值。如 Lua 内置函数 `string.find` 在查找成功后返回两个值：起始位置和结束位置。如果使用了捕获组 `"()"` 则还会返回捕获组的结果。

如果一个函数需要在 `return` 后面返回多个值，多个值间需要 `","` 隔开。

#### 5.注意

- 全局变量会占用全局名称空间，同时也有性能损耗（查询全局环境表的开销），因此应当尽量使用“局部函数”
- 函数的定义需要放置在函数调用之前

#### 6.常用函数

1. **符串操作**
   - `string.len(s)`：返回字符串 `s` 的长度
   - `string.sub(s, i, j)`：返回字符串 `s` 从第 `i` 个字符到第 `j` 个字符的子串
   - `string.find(s, pattern)`：在字符串 `s` 中查找指定的模式 `pattern`
2. **表操作**
   - `table.insert(table, [pos,] value)`：将 `value` 插入到表 `table` 中的指定位置（默认在末尾）。
   - `table.remove(table [, pos])`：从表 `table` 中移除指定位置（默认是末尾）的元素。
   - `table.concat(table [, sep [, i [, j]]])`：将表 `table` 中的元素连接成一个字符串，可以指定分隔符 `sep`
3. **数学运算**
   - `math.abs(x)`：返回 `x` 的绝对值
   - `math.sqrt(x)`：返回 `x `的平方根
   - `math.random([m [, n]])`：返回一个范围在 `[m, n]` 之间的随机数
4. **文件操作**
   - `io.open(filename [, mode])`：打开文件，返回文件句柄
   - `file:read([format])`：读取文件内容，可以指定读取的格式
   - `file:write(...)`：将数据写入文件
5. **其他常用函数**
   - `print(...)`：打印输出
   - `type(x)`：返回变量 `x` 的类型
   - `tonumber(s [, base])`：将字符串 `s` 转换为数字
6. **比较操作**
   - `==`：比较两个元素是否相等
   - `~=`：比较两个元素是否不等
   - `not x`：取 x 的反

### 2.8 Lua的面向对象

在 Lua 中使用表（`table`）实现面向对象，一个表就是一个对象。表可以拥有所有 8 大数据类型的成员属性。

下面在 DataType.lua 模块中定义带有一个成员的 `_Square` 类。

```lua
-- 正方形类
_Square = {
    size = 0
}

-- 类的方法 getArea
function _Square.getArea(self)
    return self.size * self.size;
end

-- 类的方法 new
function _Square.new(self, size)
    local cls = {}
    -- 指定 cls 的 metatable 为 _Square，所以 cls 就继承了 _Square 的属性和方法 
    setmetatable(cls, _Square)
    -- 指定方法和属性的索引表为自身
    self.__index = self
    cls.size = size or 0
    return cls;
end

square = _Square:new(3)
print(square:getArea())
```

在调用 Square 类的方法时，建议将点号改为冒号。使用冒号进行成员方法调用时，Lua 会隐形传递一个 `self` 参数，它将调用者对象本身作为第一个参数传递进来。

```lua
local dataType = require("luaScript/module/demo/dataType")
ngx.say("<br><hr>面向对象操作：<br>")
local Square = dataType.Square;
local square = Square:new(20);
ngx.say("正方形的面积为：", square:getArea());
```

这里用到了两个重要概念：

- `metadable` 元表：如果一个表（也称对象）的属性找不到，就去它的元表中查找，通过 `setmetatable(table, metatable)` 方法设置一个表的元表
- 准确来说，不是直接查找元表的属性，而是去元表中的一个特定的属性 `__index`（表）中查找属性，`__index` 也是一个 `table` 类型，Lua 会在 `__index` 中查找相应的属性

> **注意**
>
> 最好同时声明 `setmetatable(table, metatable)` 和 `self.__index = self`，否则部分 lua 版本中无法成功通过实例 `square` 调用类的方法 `getArea()`。

下面是一个继承的代码示例：

```lua
Rectangle = {
    length = 0,
    width = 0
}

function Rectangle:new(length, width)
    local obj = setmetatable({}, Rectangle)
    self.__index = self
    obj.length = length or 0
    obj.width = width or 0
    return obj
end

function Rectangle:getArea()
    return self.length * self.width
end


Square = {
    size = 0
}

function Square:new(size)
    local obj = Rectangle:new(size, size)
    self.__index = self
    obj.size = size or 0
    return obj
end

square = Square:new(20)
ngx.say(square:getArea())
ngx.say("<br>")
ngx.say(square.size)
```

## 3.Nginx Lua编程

OpenResty 汇聚了各种设计精良的 Nginx 模块，使得开发人员通过使用 Lua 脚本调动 Nginx 支持的各种 C 以及 Lua 模块，快速构造出足以胜任 10KB 乃至 1000KB 以上单机并发连接的高性能 Web 应用系统。

OpenResty 的目标是让 Web 服务直接跑在 Nginx 服务内部，充分利用 Nginx 的非阻塞 I/O 模型，不仅对 HTTP 客户端请求，甚至对远程后端（如 MySQL、PostgreSQL、Memcached 以及 Redis 等）都进行一致的高性能响应。

### 3.1 执行原理

`ngx_lua` 是将 Lua 嵌入 Nginx，让 Nginx 执行 Lua 脚本，并且高并发、非阻塞地处理各种请求。Lua 内置协程可以很好地将异步回调转换成顺序调用的形式。`ngx_lua` 在 Lua 中进行的 IO 操作都会委托给 Nginx 的时间模型，从而实现非阻塞调用。开发者可以采用串行的方式编写程序，`ngx_lua` 会在进行阻塞的 IO 操作时自动中断，保存上下文，然后将 IO 操作委托给 Nginx 事件处理机制，在 IO 操作完成后，`ngx_lua` 会恢复上下文，程序继续执行。

每个 Nginx 的 Worker 进程持有一个 Lua 解释器或 LuaJIT 实例，被这个 Worker 处理的所有请求共享这个实例。每个请求的 context 上下文会被 Lua 轻量级的协程分隔，从而保证各个请求是独立的。

其工作流程如下：

- 每个 Worker 创建一个 LuaJIT VM，Worker 内所有协程共享
- 将 Nginx I/O 原语封装后注入 Lua VM，允许 Lua 代码直接访问
- 每个外部请求都由一个 Lua 协程处理，协程之间数据隔离
- Lua 代码调用 I/O 操作等异步接口时会挂起当前协程而不阻塞 Worker 进程
- I/O 等异步操作完成时还原协程相关的上下文数据，并继续运行

`ngx_lua	` 采用 `one-coroutine-per-request` 的处理模型，对于每个请求，`ngx_lua` 会唤醒一个协程来处理用户请求，当请求处理完成后协程会被销毁。每个协程都有一个独立的全局环境，继承于全局共享的、只读的公共数据。得益于 Lua 协程的支持，`ngx_lua` 在处理 10,000 个并发请求时只需要很少的内存，一般情况下每个请求只需要 2KB 的内存，如果使用 LuaJIT 会更少，所以 `ngx_lua` 非常适合用于实现可扩展的、高并发的服务。

### 3.2 配置指令

`ngx_lua` 定义了一系列 Nginx 配置指令，用于在配置文件中配置何时运行用户 Lua 脚本以及如何返回 Lua 脚本的执行结果。

|         配置指令          | 指令说明                                                     |
| :-----------------------: | ------------------------------------------------------------ |
|    `lua_package_path`     | 配置用 Lua 外部库的搜索路径，搜索的文件类型为 `.lua` 文件    |
|    `lua_package_cpath`    | 配置用 Lua 外部库的搜索路径，搜索 C 语言编写的外部库文件；Linux 下为 `.so` 文件，Windows 下为 `.dll` 文件 |
|       `init_by_lua`       | Master 进程启动时挂载的 Lua 代码块，常用于导入公共模块       |
|    `init_by_lua_file`     | Master 进程启动时挂载的 Lua 脚本文件                         |
|   `init_worker_by_lua`    | Worker 进程启动时挂载的 Lua 代码块，常用于执行一些定时器任务 |
| `init_worker_by_lua_file` | Worker 进程启动时挂载的 Lua 脚本文件，常用于执行一些定时器任务 |
|       `set_by_lua`        | 类似于 `rewrite` 模块的`set` 指令，将 Lua 代码块的返回结果设置在 Nginx 的变量中 |
|     `set_by_lua_file`     | 类似于 `rewrite` 模块的 `set` 指令，将 Lua 脚本文件的返回结果设置在 Nginx 的变量中 |
|     `content_by_lua`      | 执行在 `content` 阶段的 Lua 代码块，执行结果将作为请求响应的内容。Lua 代码块是编写在 Nginx 字符串中的 Lua 脚本，可能需要进行特殊字符转义 |
|   `content_by_lua_file`   | 执行在 `content` 阶段的 Lua 脚本文件，执行结果将作为请求响应的内容 |
|  `content_by_lua_block`   | 与 `content_by_lua` 指令类似，不同之处在于该指令直接在一堆花括号 `"{}"` 中编写 Lua 脚本源码，而不是在 Nginx 字符串中 |
|     `rewrite_by_lua`      | 执行在 `rewrite` 阶段的 Lua 代码块，完成转发、重定向、缓存等功能 |
|   `rewrite_by_lua_file`   | 执行在 `rewrite` 阶段的 Lua 脚本文件，完成转发、重定向、缓存等功能 |
|      `access_by_lua`      | 执行在 `access` 阶段的 Lua 代码块，完成IP 准入、接口权限等功能 |
|   `access_by_lua_file`    | 执行在 `access` 阶段的 Lua 脚本文件，完成IP 准入、接口权限等功能 |
|  `header_filter_by_lua`   | 响应头部过滤处理的 Lua 代码块，比如可以用于添加响应头部信息  |
|   `body_filter_by_lua`    | 响应体过滤处理的 Lua 代码块，比如可以用于加密响应体          |
|       `log_by_lua`        | 异步完成日志记录的 Lua 代码块，比如既可以在本地记录日志，又可以记录到 ETL 集群 |

`ngx_lua` 配置指令在 Nginx 的 HTTP 请求处理阶段所处的位置如图所示

<img src="img/第03章_Nginx Lua/image-20240104012541266.png" alt="image-20240104012541266" style="zoom:67%;" />

下面介绍一些常用的配置。

#### 1.lua_package_path

```nginx
lua_package_path lua-style-path-str
```

`lua_package_path` 指令用于设置 `".lua"` 外部库的搜索路径，此指令的上下文为 `http` 配置块。它的默认值是 `LUA_PATH` 环境变量内容或者 Lua 编译的默认值。`lua-style-path-str` 字符串是标准的 lua path 格式，`";;"` 常用于表示原始的搜索路径。

```nginx
# 设置纯 Lua 扩展库的搜索路径
lua_package_path '/foo/bar/?.lua;/blah/?.lua;;';
```

OpenResty 可以在搜索路径中使用插值变量。例如可以使用插值变量 `$prefix` 或 `${prefix}` 获取虚拟服务器 server 的前缀路径，server 的前缀路径通常在 Nginx 服务器启动时通过 `-p PATH` 命令行选项来指定。

#### 2.lua_package_cpath

```nginx
lua_package_cpath lua-style-cpath-str
```

`lua_package_cpath` 指令用于设置 Lua 的 C 语言模块外部库 `".so"` 或 `".dll"` 的搜索路径，上下文为 `http` 配置块。`lua-style-cpath-str` 字符串是标准的 lua cpath 格式，`";;"` 用于表示原始的 cpath。

```nginx
# 设置 C 编写的 Lua 扩展模块的搜索路径
lua_package_pcath '/foo/baz/?.so;/blah/?.so;;';
```

同样支持 `$prefix` 和 `${prefix}`。

#### 3.init_by_lua

```nginx
init_by_lua lua-script-str
```

`init_by_lua` 指令只能用于 `http` 上下文中，运行在配置加载阶段。当 nginx 的 master 进程在加载 Nginx 配置文件时，在全局 Lua VM 级别上运行由参数 `lua-script-str` 指定的 Lua 脚本块。当 Nginx 接收到 HUP 信号并开始重新加载配置文件时，Lua VM 将会被重新创建，并且 `init_by_lua` 将在新的 VM 上再次运行。

如果 Lua 脚本的缓存是关闭的，那么每一次请求都运行一次 `init_by_lua` 处理程序，通过 `lua_code_cache` 指令可以关闭 Lua 脚本缓存，默认是开启的。

> **注意**
>
> 在生产环境下都会开启 Lua 脚本缓存，在 `init_by_lua` 调用 `require` 所加载的模块文件会缓存在全局的 Lua 注册表 package.loaded 中，所以在这里定义的全局变量和函数可能会污染命名空间，当然也会影响性能。

#### 4.lua_code_cache

```nginx
lua_code_cache on（默认） | off
```

`lua_code_cache` 用于启用或禁用 Lua 脚本缓存，可以使用的上下文有 `http`、`server`、`location`。当关闭缓存时，通过 `ngx_lua` 提供的每个请求都将在一个单独的 Lua VM 实例中运行。在缓存关闭的情况下，在 `set_by_lua_file`、`content_by_lua_file`、`access_by_lua_file` 等指令中引用的 Lua 脚本都将不会被缓存，所有的 Lua 脚本都从新加载。

但是通过该指令，开发人员可以进行快速开发，该动代码后不需要重启 Nginx。关闭缓存对整体性能会产生负面的影响。如在禁用 Lua 脚本缓存后，一个简单的 “hello world” 示例的性能可能会下降一个数量级。

在生产环境中必须开启 Lua 脚本缓存，仅仅可以在开发期间关闭。

#### 5.set_by_lua

```nginx
set_by_lua $destVar lua-script-str params
```

`set_by_lua` 指令的功能类似于 `rewrite` 模块的 `set` 指令，是将 Lua 脚本块的返回结果设置在 Nginx 的变量中。`set_by_lua` 指令所处的上下文和执行阶段与 Nginx 的 `set` 指令基本相同。

使用 `set_by_lua` 配置指令时，可以在 Lua 脚本的后面带上一个调用参数列表。在 Lua 脚本中可以通过 Nginx Lua 模块内置的 `ngx.arg[n]` 读取实际参数。

下面是一个简单的示例，将 Lua 脚本的相加结果设置给 Nginx 的变量 `$sum`，具体代码如下：

```nginx
location /set_by_lua_demo {
    set $foo 1;
    set $bar 2;
    
    # 调用內联代码，将结果放入 Nginx 变量 $sum
    set_by_lua $sum 'return tonumber(ngx.arg[1]) + tonumber(ngs.arg[2])' $foo $bar;
    echo ”$foo + $bar = $sum“;
}
```

在这段代码中，`set_by_lua` 将两个输入参数 `$a`、`$b` 累积起来，然后将相加的结果设置到 Nginx 的变量 `$sum` 中。

#### 6.access_by_lua

```nginx
access_by_lua $destVar lua-script-str
```

`access_by_lua` 执行在 HTTP 请求处理 11 个阶段的 `access` 阶段，使用 Lua 脚本进行访问控制。`access_by_lua` 指令运行于 `access` 阶段的末尾，因此总是在 `allow` 和 `deny` 这样的指令之后运行。一般可以通过 `access_by_lua` 进行比较复杂的用户权限验证，因为能借助 Lua 脚本执行一系列复杂的验证操作，比如实时查询数据库或者其他后端服务。

下面的示例中利用 `access_by_lua` 实现 `ngx_access` 模块的 IP 地址过滤功能：

```nginx
location /access_demo {
    access_by_lua '
        ngx.log(ngx.DEBUG, "remote_addr = " .. ngx.var.remote_addr)
        if ngx.var.remote_addr == "133.203.55.21" then
            return
        end
            ngx.exit(ngx.HTTP_UNAUTHORIZED)
       ';
    echo "hello world";
}
```

这段代码检测远程 IP 地址是否为 133.203.55.21，如果请求来源不是这个就通过 `ngx_lua` 提供的 `ngx.exit()` 中断当前的整个请求处理流程，直接返回 401（未授权错误）。如果 `access-by_lua` 指令没有将 HTTP 请求处理流程中断，处于 `access` 阶段后面的 `content` 阶段就会顺利执行，`echo` 指令的结果就能输出给客户端。

#### 7.content_by_lua

```nginx
content_by_lua lua-script-str
```

`content_by_lua` 用于设置执行在 `content` 阶段的 Lua 代码块，执行结果将作为请求响应的内容。该指令可以用于 `location` 上下文。

`lua-script-str` 代码块用于在 Nginx 配置文件中编写字符串形式的 Lua 脚本，可能需要进行特殊字符转义，所以现在**不鼓励使用该命令**，改为使用 `content_by_lua_block` 指令代替。`content_by_lua_block` 指令代码块使用花括号 `"{}"` 定义，不再使用字符串分隔符。

### 3.3 常用功能

#### 1.获取URL中的参数

下面的示例通过 Lua 脚本获取 URL 中的 query 参数然后进行相加。

```nginx
location /add_params_demo {
    set_by_lua $sum '
        local args = ngx.req.get_uri_args()
        local a = args["a"]
        local b = args["b"]
        return a + b
    ';
    echo "$arg_a + $arg_b = $sum";
}
```

除了通过 `ngx.req.get_uri_args()` 获取参数外，还可以通过 Nginx 内置变量 `$arg_PARAMETER` 获取请求参数的值，然后传递给 `set_by_lua` 指令：

```nginx
location /add_params_demo {
    set_by_lua $sum "
        local a = tonumber(ngx.arg[1])
        local b = tonumber(ngx.arg[2])
        return a + b;
    " $arg_a $arg_b;
    echo "$arg_a + $arg_b = $sum";
}
```

#### 2.设置响应头

可通过内置变量 `ngx.header` 来访问和设置 HTTP 响应头字段，`ngx.header` 的类型为 `table`，可以通过 `ngx.header.HEADER` 形式引用某个响应头。一个示例如下：

```nginx
content_by_lua_block {
    ngx.header["header1"] = "value1"
    ngx.header.header2 = 2
    ngx.header.set_cookie = {'Foo=bar; test=ok; path=/', 'age=18; path=/'}
    ngx.say("check header")
}
```

以上代码设置了响应头 `header1` 的值为 `value1`、`header2` 的值为 2；`ngx.header.set_cookie` 变量用于设置响应头 `set-cookie` 的值，使用 `table` 类型的对象可以一次设置多个。

响应结果如下：

```bash
HTTP/1.1 200 OK
Connection: keep-alive
Content-Type: text/html; charset=utf-8
Date: Thu, 04 Jan 2024 14:40:31 GMT
header1: value1
header2: 2
Server: openresty/1.21.4.3
Set-Cookie: Foo=bar; test=ok; path=/
Set-Cookie: age=18; path =/
Transfer-Encoding: Identity
```

Cookie 是通过请求的 `set-cookie` 响应头来保存的，HTTP 响应内容中可以包含多个 `set-cookie` 响应头，一个 `set-cookie` 的值通常是一个字符串，该字符串大致包含下表所示的信息和属性（不区分大些小）：

| Cookie 信息（或属性） | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| Cookie 名称           | Cookie 名称的组成字符只能使用可以用于 URL 中的字符，一般为字母和数字，不能包含特殊字符，若有特殊字符需要进行转码。Cookie 名称为 Cookie 字符串的第一组键值对中的 key |
| Cookie 值             | Cookie 值的字符组成规则和 Cookie 名称相同， 若有特殊字符则需要进行转码。Cookie 值为 Cookie 字符串的第一组键值对中的 value |
| expires               | Cookie 过期日期，这是一个 GMT 格式的时间，当过了这个日期之后，浏览器就会将这个 Cookie 删除掉，不设置 expires 时 Cookie 在浏览器关闭后消失 |
| path                  | Cookie 的访问路径，此属性设置指定路径下的页面才可以访问该 Cookie。访问路径的值一般设为 `"/"`，表示同一个站点的所有页面都可以访问这个 Cookie |
| domain                | Cookie 的访问域名，此属性设置指定域名下的页面才可以访问该 Cookie。例如要让 Cookie 只能在 a.test.com 域名才可以访问，可将其 domain 设置为 a.test.com |
| Secure                | Cookie 的安全属性，此属性设置该 Cookie 是否只能通过 HTTPS 协议访问。一般的 Cookie 使用 HTTP 即可访问，如果设置了 Secure 属性（没有属性值），则只有使用 HTTPS 协议 Cookie 才可以被访问 |
| HttpOnly              | 如果 Cookie 设置了 HttpOnly 属性，那么通过程序（JS 脚本、Applet 等）将无法读取到 Cookie 信息，可以防止 XSS 攻击。HttpOnly 属性和 Secure 一样都没有值，只有名称 |

为了通信安全，某些场景下只能在前后端之间使用 HTTPS 协议通信，如微信小程序的官网要求必须使用 HTTPS 协议。这种场景下，在内网环境可以继续使用 HTTP 通信协议，然后通过 Nginx 网关完成外部 HTTPS 协议到内部 HTTP 协议的转换。此时 Nginx 外部网关可以对 Cookie 属性进行修改，增加 `Secure` 安全属性。

此外，大部分场景下确实不需要在前端脚本中访问 Cookie。Cookie 信息仅仅在后端 Java 容器中访问（如 Session 会话 ID），不需要在前端 JS 等脚本中访问，此时可以对 Cookie 属性进行修改，增加 `HttpOnly` 安全属性，这将有助于缓解跨站点脚本攻击。

为 Cookie 增加 `HttpOnly` 安全属性的操作可以通过 Servlet 过滤器的形式在 Java 容器中完成：

```java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain) throws IOException, ServletException {
    HttpServletRequest req = (HttpServletRequest) request;
    HttpServletResponse resp = (HttpServletResponse) response;
    Cookie[] cookies = req.getCookies();
    if (cookies != null) {
        for (Cookie : cookies) {
            String name = cookie.getName();
            String value = cookie.getValue();
            StringBuilder sb = new StringBuilder();
            sb.append(name).append("=").append(value).append("; httpOnly "); // 在 cookie 末尾添加 httpOnly 安全属性
            resp.setHeader("Set-Cookie", sb.toString());
        }
    }
    filterChain.doFilter(request, response);
}
```

更好的方式是在反向代理外部网关 Nginx 中完成：

```nginx
# 模拟上游服务器
location /header_demo {
    content_by_lua_block {
        ngx.header["header1"] = "value1"
        ngx.header.header2 = 2
        ngx.header.set_cookie = {'Foo=bar; test=ok; path=/', 'age=18; path=/'}
        ngx.say("header demo")
    }
}

# 反向代理外部网关
location /header_filter_demo {
    proxy_pass http://127.0.0.1/header_demo;
    
    header_filter_by_lua_block {
        local cookies = ngx.header.set_cookie
        if cookies then
            if type(cookies) == "table" then
            	local cookie = {}
        		for k, v in pairs(cookies) do
            		cookie[k] = v .. "; secure; httpOnly"
            	end
            	ngx.header.set_cookie = cookie
            else
            	ngx.header.set_cookie = cookies .. "; secure; httpOnly"
            end
        end
    }
}
```

访问 `/header_filter_demo` 查看返回值的 Cookie 头：

<img src="img/第03章_Nginx Lua/image-20240105215804884.png" alt="image-20240105215804884" style="zoom:67%;" />

可以看到 Foo 和 age 两个 Cookie 被标记为了不安全，因为没有通过 HTTPS 协议访问。

#### 3.访问Nginx变量

不论是在[核心模块内置变量](第02章_基础使用.md#6.核心模块内置变量)中介绍的 Nginx 的内置变量，还是在配置文件中使用 `set` 指令定义的 Nginx 变量，都可以在 Lua 代码中通过 `ngx.var` 进行访问。

```nginx
location /lua_var_demo {
    set $hello world;
    
    content_by_lua_block {
        local basic = require("luaScript.module.common.basic");
        
        local vars = {};
        -- 访问内置变量
        vars.remote_addr = ngx.var.remote_addr;
        vars.request_uri = ngx.var.request_uri;
        vars.query_string = ngx.var.query_string;
        vars.uri = ngx.var.uri;
        vars.nginx_version = ngx.var.nginx_version;
        vars.server_protocol = ngx.var.server_protocol;
        vars.remote_user = ngx.var.remote_user;
        vars.request_filename = ngx.var.request_filename;
        vars.request_method = ngx.var.request_method;
        vars.document_root = ngx.var.document_root;
        vars.body_bytes_sent = ngx.var.body_bytes_sent;
        vars.binary_remote_addr = ngx.var.binary_remote_addr;
        vars.args = ngx.var.args;
        
        -- 访问自定义变量
        vars.hello = ngx.var.hello;
        
        -- 访问请求参数
        vars.foo = ngx.var.arg_foo;
        
        local str= basic.tableToStr(vars, "<br>");
        ngx.say(str);
    }
}
```

其中的 `tableToStr()` 方法如下

```lua
local function tableToStr(vars, delimiter)
    string = "";
    for k, v in pairs(vars) do
        string = string .. "[" .. k .. "] = '" .. v .. "'" .. delimiter
    end
    return string;
end
```

结果如下

```bash
[remote_addr] = '133.203.55.21'
[request_uri] = '/lua_var_demo'
[uri] = '/lua_var_demo'
[server_protocol] = 'HTTP/1.1'
[request_method] = 'GET'
[request_filename] = './html/lua_var_demo'
[nginx_version] = '1.21.4'
[document_root] = './html'
[body_bytes_sent] = '0'
[binary_remote_addr] = '��7'
[hello] = 'world'
```

#### 4.访问请求上下文变量

Nginx 执行 Lua 脚本涉及很多阶段，每个阶段都可以嵌入不同的 Lua 脚本，不同阶段的 Lua 脚本可以通过 `ngx.ctx` 进行上下文变量的共享。

`ngx.ctx` 上下文实质上是一个 Lua `table`，生存周期与当前请求相同，当前请求不同阶段嵌入的 Lua 脚本都可以读写 `ngx.ctx` 表中的属性。

```nginx
location /ctx_demo {
    rewrite_by_lua_block {
        ngx.ctx.var1 = 1;
    }
    
    access_by_lua_block {
        ngx.ctx.var2 = 2;
    }
    
    content_by_lua_block {
        local basic = require("luaScript.module.common.basic");
        ngx.ctx.var3 = 3;
        local result = ngx.ctx.var1 + ngx.ctx.var2 + ngx.ctx.var3;
        ngx.ctx.sum = result;
        local str = basic.tableToStr(ngx.ctx, "<br>");
        ngx.say(str);
    }
}
```

结果如下

```bash
[sum] = '6'
[var2] = '2'
[var1] = '1'
[var3] = '3'
```

可以看出，`ngx.ctx` 表中定义的属性可以在请求处理的 `rewrite`、`access`、`content` 等处理阶段进行共享。需要注意的是，在 `ngx_lua` 模块中，每个请求（包括子请求）都有一份独立的 `ngx.ctx` 表。

## 4.重定向与内部子请求

Nginx 的 `rewrite` 指令不仅可以在 Nginx 内部的 `server`、`location` 之间进行跳转，还可以进行外部链接的重定向。通过 `ngx_lua` 模块的 Lua 函数除了能实现 Nginx 的 `rewrite` 指令外，还能完成内部子请求、并发子请求等复杂功能。

### 4.1 重定向

`ngx_lua` 可以实现 Nginx 的 `rewrite` 指令类似的功能，该模块提供了两个 API 来实现重定向的功能：

- `ngx.exec(uri, args?)`：内部重定向
- `ngx.redirect(uri, status?)`：外部重定向

> **注意**
>
> 不论 `ngx.exec(...)` 还是 `ngx.redirect(...)` 都==**不会自动传递原始参数**==。

#### 1.内部重定向

`ngx.exec()` 等价于 `rewrite regrex replacement last`，下面是三个示例：

```lua
-- 重定向到 /internal/sum
ngx.exec('/internal/sum');

-- 重定向到 /internal/sum?a=3&b=5，并且追加参数 c=6
ngx.exec('/internal/sum?a=3&b=5', 'c=6');

-- 重定向到 /internal/sum，并且追加参数 ?a=3&b=5&c=6
ngx.exec('/internal/sum', {a=3, b=5, c=6});
```

下面是一个完整的示例，通过内部重定向完成 3 个参数的累加：

```nginx
location /internal/sum {
    internal; # 只允许内部调用
    content_by_lua_block {
        local arg_a = tonumber(ngx.var.arg_a);
        local arg_b = tonumber(ngx.var.arg_b);
        local arg_c = tonumber(ngx.var.arg_c);
        local sum = arg_a + arg_b + arg_c;
        ngx.say(arg_a, "+", arg_b, "+", arg_c, "=", sum);
    }
}

location /sum1 {
    content_by_lua_block {
        return ngx.exec("/internal/sum", {a = 100, b = 10, c = 1});
    }
}

location /sum2 {
    # 等同于以下 rewrite 指令
    rewrite /sum2 /internal/sum?a=100&b=10&c=1 last;
}
```

> **注意**
>
> - 如果有 `args` 参数，参数可以是字符串的形式，也可以是 Lua `table` 的形式：
>
>   ```lua
>   ngx.exec("/internal/sum", "a=100&b=5");
>   ngx.exec("/internal/sum", {a=100, b=5});
>   ```
>
>   第二个参数可以传递 `ngx.req.get_uri_args()` 将原始 query 参数传递下去。
>
> - 该方法可能不会主动返回，建议在调用时加上 `return`
>
>   ```nginx
>   return ngx.exec(...)
>   ```
>

#### 2.外部重定向

```lua
ngx.redirect(uri, status?)
```

`ngx.redirect` 外部重定向方法与 `ngx.exec` 内部重定向方法不同，外部重定向将通过客户端进行二次跳转，所以 `ngx.redirect` 方法会产生额外的网络流量，该方法的第二个参数为响应状态码，可以传递 `301/302/303/307/308` 重定向状态码。其中 301、302 是 HTTP 1.0 协议定义的相应码，302、307、308 是 HTTP 1.1 协议定义的相应码。

如果不指定 `status`，那么默认的相应状态为 `302 (ngx.HTTP_MOVED_TEMPORARILY)` 临时重定向。

```nginx
# 注意取消 /internal/sum 的 internal 属性
location /sum3 {
    content_by_lua_block {
        return ngx.redirect("/internal/sum?a=100&b=10&c=1")
    }
}

location /sum4 {
    rewrite ^/sum4 "/internal/sum?a=100&b=10&c=1" redirect;
}
```

如果指定 `status` 为 301，那么对应的常量为 `ngx.HTTP_MOVED_PERMANENTLY` 永久重定向，对应到 `rewrite` 指令的标志位为 `permanent`。

```nginx
location /sum5 {
    content_by_lua_block {
        return ngx.redirect("/internal/sum?a=100&b=10&c=1", ngx.HTTP_MOVED_PERMANENTLY);
    }
}

location /sum6 {
    rewrite ^/sum6 "/internal/sum?a=100&b=10&c=1" permanent;
}
```

下面是一个综合案例，通过 `ngx.redirect` 和 `rewrite` 进行 3 种方式的外部跳转：

```nginx
# ngx_lua 中获取 location 中的匹配组
location ~*/baidu1/(.*) {
    content_by_lua_block {
        return ngx.redirect("https://www.baidu.com/" .. ngx.var[1])
    }
}

# ngx_lua 中获取 rewrite 指令中的匹配组
location ~*/baidu2/.* {
    rewrite ^/baidu2/(.*) $1 break;
    content_by_lua_block {
            return ngx.redirect("https://www.baidu.com/" .. ngx.var[1])
    }
}

# rewrite 中获取自身的匹配组，会自动传递 query 参数
location ~*/baidu3/.* {
    rewrite ^/baidu3/(.*) https://www.baidu.com/$1 redirect;
}
```

访问 `/baidu[n]/s?wd=test` 发现只有 `baidu3` 传递了参数，其他两个只二次跳转到了 `www.baidu.com/s`。可以通过以下方式获取参数列表拼接到跳转地址后面：

```nginx
location ~*/baidu1/(.*) {
    content_by_lua_block {
        function parseParams(args)
            str = "?"
            for k, v in pairs(args) do
                str = str .. k .. "=" .. v .. "&"
            end
            return str
        end
        return ngx.redirect("https://www.baidu.com/" .. ngx.var[1] .. parseParams(ngx.req.get_uri_args()))
    }
}
```

当然实际使用时要把 `parseParams()` 抽取出来。

> **注意**
>
> 和 `exec` 一样，`ngx.redirect` 不会主动返回，需要显示调用 `return`。

###  4.2 子请求

`Nginx` 子请求不是 HTTP 协议的标准实现，是 nginx 特有的设计，主要为了提高内部对单个客户端请求处理的并发能力。

如果某个客户端的请求访问了多个内部资源，为了提高效率，可以为每一个内部资源访问建立单个子请求，并让所有子请求同时进行。

子请求并不是由客户端直接发起的，它是由 Nginx 服务器在处理客户端请求时根据自身逻辑需要而内部建立的新请求。因此，子请求只在 Nginx 服务器内部进行处理，不会与客户端进行交互。通常情况下，为保护子请求所定义的内部接口，会把这些接口设置为 `internal`，防止外部直接访问。

发起单个子请求可以使用 `ngx.location.capture()` 方法，格式如下：

```lua
ngx.location.capture(uri, options?)
```

`capture` 方法的第二个参数 `options` 是一个 `table`，用于设置子请求相关的选项，又如下可以设置的选项：

- `method`：子请求的方法，默认为 `ngx.HTTP_GET`
- `body`：传给子请求的请求体，仅限于 `string` 或 `nil`
- `args`：传给子请求的请求参数，支持 `string` 或 `table`
- `vars`：传给子请求的变量表，仅限于 `table`
- `ctx`：父子请求共享的变量表 `table`
- `copy_all_vars`：复制所有变量给子请求
- `share_all_vars`：父子请求共享所有变量
- `always_forward_body`：用于设置是否转发请求体

下面是一个综合性实例，包含两个请求接口，外部接口供外部访问使用，在准备好必要的请求参数、上下文环境变量、请求体之后，调用内部访问接口获取执行结果，然后返回给客户端。具体如下：

```nginx
# 外部访问接口
location ~ /goods/detail/([0-9]+) {
    set $goodsId $1;
    set $var1 '';
    set $var2 '';
    content_by_lua_block {
        -- 解析 body 之前一定要先读取 request body
        ngx.req_read_body()
        local uri = "/internal/detail/" .. ngx.var.goodsId
        local request_method = ngx.var.request_method
        local args = ngx.req.get_uri_args()
        local shareCtx = {c1 = "parent value", other = "other value"}
        local res = ngx.location.capture(uri, {
            method = ngx.HTTP_GET,
            args = args,
            body = 'customed request body',
            vars = {
                var1 = "value1",
                var2 = "value2"
            },
            -- 转发父请求的 request body
            always_forward_body = true
            ctx = shareCtx
        })
        ngx.say("child res.status:", res.status)
        ngx.say(res.body)
        ngx.say("<br>shareCtx.c1=", shareCtx.c1)
    }
}
```

```nginx
# 内部模拟上游服务接口
location ~ /internal/detail/([0-9]+) {
    internal;
    set $goodsId $1;
    
    content_by_lua_block {
        ngx.req.read_body()
        ngx.say("<br><hr>child scope: <br>")
        -- 访问父请求传递的参数
        local args = ngx.req.get_uri_args()
        ngx.say("<br>foo=", args.foo)
        -- 访问父请求传递的请求体
        local data = ngx.req.get_body_data()
        ngx.say("<br>data=", data)
        -- 访问 Nginx 定义的变量
        ngx.say("<br>goodsId=", ngx.var.goodsId)
        -- 访问父请求传递的变量
        ngx.say("<br>var.var1=", ngx.var.var1)
        -- 访问父请求传递的共享上下文，并修改其属性
        ngx.say("<br>ngx.ctx.c1=", ngx.ctx.c1)
        ngx.say("<br>child end<hr>")
        ngx.ctx.c1 = "changed value by child"
    }
}
```

访问 `http://43.153.170.51/goods/detail/10?foo=foo` 结果如下：

```bash
child res.status:200
child scope:
foo=foo
data=customed request body
goodsId=10
var.var1=value1
ngx.ctx.c1=v1
child end
shareCtx.c1=changed value by child
```

**`capture` 的参数**

- `capture` 的第二个参数 `options` 是一个 `table` 容器，用于设置子请求的选项。

- `method`：用于指定子请求的 `methid` 类型。

  ```lua
  local res = ngx.location.capture(uri, {
          method = ngx.HTTP_PUT        
      }
  )
  ```

- `args`：指定子请求的 URI 请求参数，可以是字符串或者 `table`。

  ```lua
  local res = ngx.location.capture(uri, {
          -- 将父请求的参数传递给子请求
          args = ngx.req.get_uri_args()
      }
  )
  ```

- `vars`：设置传递给子请求中的 Nginx 变量，`table` 类型。

  ```nginx
  set $var1 '';
  set $var2 '';
  content_by_lua_block {
      local res = ngx.location.capture(uri, {
              vars = {
                  var1 = "value1",
                  var2 = "value2"
              }
          }
      )
  }
  ```

  > **注意**
  >
  > 使用 `vars` 向子请求中传递 Nginx 变量时，**变量需要提前定义**，否则抛出==变量未定义==的错误。

- `ctx`：指定一个 `table` 作为子请求的 `ngx.ctx` 表。

  ```lua
  local c = {
      c1 = "v1",
      other = "other value"
  }
  local res = ngx.location.capture(uri, {
  		ctx = c;        
      }
  );
  ```

  父请求如果修改了 `ctx` 表中的成员，那么子请求可以通过 `ngx.ctx` 获取，反之，子请求也可以修改 `ngx.ctx` 中的成员，父请求可以通过 `ctx` 表获取。通过 `ctx` 属性值可以方便地让父请求和子请求进行上下文变量共享。

- `always_forward_body`：设置是否转发请求体。但设置为 true 时，父请求中的请求体 request body 将转发到子请求。

  ```lua
  local res = ngx.location.capture(uri, {
  		method = ngx.HTTP_POST,
          always_forward_body = true
      }
  )
  ```

> **注意**
>
> `ngx.location.capture` 只能发起到当前 Nginx 服务器的==内部路径==的子请求，如果需要发起外部 HTTP 路径的子请求，就需要与 `location` 反向代理配置配合实现。

### 4.3 并发子请求

一次客户端请求往往调用多个微服务接口才能获取到完整的页面内容，此时就可以通过 Nginx 进行上游接口合并。在 `OpenResty` 中 `ngx.location.capture_multi` 可以完成内部多个子请求和并发访问。

```lua
ngx.location.capture_multi({
        {uri, options?},
        {uri, options?},
        ...
    }
)
```

`capture_multi` 可以一次发送多个内部子请求，每个子请求的参数使用方式和 `capture` 方法相同。调用 `capture_multi` 前可以把所有子请求加入一个 `table` 容器作为调用参数传入。`capture_multi` 返回后可以将其结果再用 `"{}"` 包装成一个 `table`。

```nginx
# 发起两个子请求

location /capture_multi_demo {
    content_by_lua_block {
        local postBody = ngx.encode_args({
            post_k1 = 32,
            post_k2 = "post_v2"
        });
        local reqs = {};
        -- 向 reqs 中插入数据
        table.insert(reqs, {
            "/print_get_param",
            {args = "a=3&b=4"}
        });
        table.insert(reqs, {
            "/print_post_param",
            {
                method = ngx.HTTP_POST,
                args = "a=3&b=4",
                body = postBody
            }
        });
        -- 将结果封装在 table 中
        local resps = {ngx.location.capture_multi(reqs)};

        for i, res in pairs(resps) do
            ngx.say("child res.status:", res.status, "<br>")
            ngx.say("child res.body:", res.body, "<br>")
        end
    }
}
```

两个内部接口用于模拟上游的服务（如 Java 微服务），客户端是不能直接访问内部接口的。

```nginx
location /print_get_param {
    internal;
    content_by_lua_block {
        ngx.say("<br>child start:");
        local arg = ngx.req.get_uri_args();
        for k, v in pairs(arg) do
            ngx.say("<br>[GET] key:", k, " v:", v);
        end
        ngx.say("<br>child end<hr>");
    }
}

location /print_post_param {
    internal;
    content_by_lua_block {
        ngx.say("<br>child start:");
        -- 解析 body 之前一定要先读取
        ngx.req.read_body();
        local arg = ngx.req.get_post_args();
        for k, v in pairs(arg) do
            ngx.say("<br>[POST] key:", k, " v:", v);
        end
        ngx.say("<br>child end<hr");
    }
}
```

结果如下：

```bash
child res.status:200
child res.body:
child start:
[GET] key:b v:4
[GET] key:a v:3
child end

child res.status:200
child res.body:
child start:
[POST] key:post_k1 v:32
[POST] key:post_k2 v:post_v2
child end
```

在所有子请求终止之前，`capture_multi` 函数不会返回。此函数的耗时是单个子请求的最长延迟。

## 5.Nginx Lua操作Redis

【[官方文档](https://github.com/openresty/lua-resty-redis#connect)】

### 5.1 CURD基本操作

使用 Lua 模块 lua-resty-redis 之前需要确保已经导入相关的库文件（默认存在），可以在官网下载 resty/redis.lua 库文件，然后将该库文件加入项目工程所在的 Lua 外部库路径。大部分的 Redis 操作命令都实现了同名的 Lua API 方法。

下面是一个简单的示例：

```nginx
location /redis_demo {
    content_by_lua_file luaScript/module/demo/redisDemo.lua;
}
```

```lua
-- luaScript/module/demo/redisDemo.lua

local redis = require "resty.redis"
local config = require("luaScript.module.config.redis-config")
-- 创建 redis 实例
local red = redis:new()

-- 设置超时时间: connect_timeout, send_timeout, read_timeout
red:set_timeouts(config.timeout, config.timeout, config.timeout)

-- 连接服务器
local ok, err = red:connect(config.host_name, config.port)
if not ok then
    ngx.say("failed to connect: ", err)
    return
end

-- 密码
local ok, err = red:auth(config.password)
if not ok then
    ngx.say("failed to auth: ", err)
    return
end

-- 选择数据库
local ok, err = red:select(config.db)
if not ok then
    ngx.say("failed to select db: ", err)
    return
end

-- 设置值
ok, err = red:set("animal", "dog")
if not ok then
    ngx.say("failed to set dog: ", err, "<br>")
    return
else
    ngx.say("set dog: ok", "<br>")
end

-- 取值
local res, err = red:get("animal")
-- 判空
if not res or res == ngx.null then
    ngx.say("failed to get dog: ", err, "<br>")
    return
else
    ngx.say("get dog: ok - ", res, "<br>")
end

-- 批量操作，减少网络 IO 次数
red:init_pipeline()
red:set("cat", "cat 1")
red:set("horse", "horse 1")
red:get("cat")
red:get("horse")
red:get("animal")
local results, err = red:commit_pipeline()
if not results then
    ngx.say("failed to commit the pipeline requests: ", err)
    return
end

-- 处理结果
for i, res in ipairs(results) do
    if type(res) == "table" then
        if res[1] == false then
            ngx.say("failed to run commend ",i, ": ", res[2], "<br>")
        else
            ngx.say("successed to run commend ", i, ": ", res[2], "<br>")
        end
    else
        ngx.say("successed to run commend ", i, ": ", res, "<br>")
    end
end

red:close()
```

导入的 redid-config.lua 配置文件如下：

```lua
local _module = {
    -- 服务器地址
    host_name = "10.244.0.254",
    -- 服务器端口
    port = "6379",
    -- 服务器数据库
    db = "1",
    -- 服务器密码
    password = "123456",
    -- 超时时间
    timeout = 20000,
    -- 线程池数量
    pool_size = 100,
    -- 最大空闲时间
    pool_max_idle_time = 10000
}

return _module
```

访问 redis_demo 结果如下：

```bash
set dog: ok
get dog: ok - dog
successed to run commend 1: OK
successed to run commend 2: OK
successed to run commend 3: cat 1
successed to run commend 4: horse 1
successed to run commend 5: dog
successed to close redis
```

### 5.2 封装一个操作基础类

RedisOperator.lua

```lua
local redis = require "resty.redis"
local config = require("luaScript.module.config.redis-config")

--输出到日志文件
local function error(string)
    if  type(string)=="string" then
        ngx.log(ngx.ERR, string);
        return
    end
    if  type(string)=="table" then
        ngx.log(ngx.ERR, table.concat(string, " "));
        return
    end
    ngx.log(ngx.ERR, tostring(string));
end

-- 统一的模块对象
local _Module = {}

-- 类的方法
function _Module:new()
    local obj = setmetatable({}, _Module)
    self.__index = self
    obj.red = nil
    return obj
end

-- 获取 redis 连接
function _Module:open()
    local red = redis:new()
    red:set_timeouts(config.timeout, config.timeout, config.timeout)
    local ok, err = red:connect(config.host_name, config.port)
    if not ok then
        error("连接 redis 服务器失败: ", err)
        return false;
    end
    if config.password then
        red:auth(config.password)
    end
    if config.db then
        red:select(config.db)
    end
    self.red = red;
    return true;
end

-- 缓存值
function _Module:setValue(key, value)
    local ok, err = self.red:set(key, value)
    if not ok then
        error("redis 缓存设置失败: ", err)
        return false;
    end
    return true;
end

-- 值递增
function _Module:incrValue(key)
    local newVal, err = self.red:incr(key)
    if not newVal then
        error("redis 缓存递增失败: ", err)
        return false;
    end
    return newVal;
end

-- 过期
function _Module:expire(key, seconds)
    local ok, err = self.red:expire(key, seconds)
    if not ok then
        error("redis 设置过期失败: ", err)
        return false;
    end
    return true;
end

-- 获取值
function _Module:getValue(key)
    local resp, err = self.red:get(key)
    if not resp then
        return nil;
    end
    return resp
end

function _Module:getSmembers(key)
    local resp, err = self.red:smembers(key)
    if not resp then
        error("redis 缓存读取失败 ")
        return nil;
    end
    return resp;
end

--缓存值
function _Module:hsetValue(key, id, value)
    local  ok, err = self.red:hset(key, id, value)
    if not ok then
        error("redis hset 失败 ")
        return false;
    end
    print("set result: ", ok)

    return true;
end

--获取值
function _Module:hgetValue(key, id)
    local resp, err = self.red:hget(key, id)
    if not resp then
        error("redis hget 失败 ")
        return nil;
    end
    return resp;
end

--执行脚本
function _Module:evalsha(sha, key1, key2)
    local resp, err = self.red:evalsha(sha, 2, key1, key2)
    if not resp then
        error("redis evalsha 执行失败 ")
        return nil;
    end
    return resp;
end

--执行秒杀的脚本
function _Module:evalSeckillSha(sha, method, goodId, userId, token)
    local resp, err = self.red:evalsha(sha, 1, method, goodId, userId, token);
    if not resp then
        error(" redis evalsha 秒杀 执行失败 ".. method .." " .. skuId .." ".. userId .." ".. token .." ".. err .." ")
        return nil;
    end
    return resp;
end

function _Module:getConnection()
   return  self.red;
end

-- 将连接还给连接池
function _Module:close()
    if not self.red then
        return
    end
    local ok, err = self.red:set_keepalive(config.pool_max_idle_time, config.pool_size)
    if not ok then
        error("连接释放失败: ", err)
    end
end

return _Module
```

使用

```lua
local redis = require "luaScript.module.demo.redisOperator"
local red = redis:new()
red:open()
red:setValue("number", "1")
red:incrValue("number")
red:expire("number", 10000)
ngx.say(red:getValue("number"))
red:close()
```

### 5.3 连接池

默认情况下每使用一次 `redis:connect()` 都会创建一个新的 Redis 连接，每一次传输数据都需要经历创建连接、收发数据、销毁连接，高并发场景下会导致以下问题：

- 连接资源被快速消耗
- 网络一旦抖动，会产生大量 TIME_WAIT 连接，需要定期重启服务程序或机器
- 服务器工作不稳定，QPS 忽高忽低

此时我们可以使用长连接，并通过一个连接池将所有长连接缓存和管理起来。在 OpenResty 中，lua-resty-redis 模块管理了一个连接池，定义了 `set_keepalive` 方法完成连接的回收和复用。

```lua
-- max_idle_timeout: 指定连接在池中的最大空闲时常（毫秒）
-- pool_size: 指定每个 Nginx 工作进程的连接池的最大连接数
ok, err = red:set_keepalive(max_idle_timeout, pool_size)
```

该方法将当前的 Redis 连接立即放入连接池。如果入池成功就返回 1，否则返回 nil 和错误描述字符串。

需要注意的是：

- 只有数据传输完毕，Redis 连接使用完成后才能调用 `set_keepalive` 方法将连接放到池子中，`set_keepalive` 会立即将 red 连接对象转换到 closed 状态，后面再调用时对象方法时会出错。
- 如果设置错误，那么 red 连接对象不一定可用，不能把可用性存疑的连接对象放到池子中

`set_keepalive` 方法完成连接回收后，下次通过 `red:connect()`  获取连接时，`connect` 方法会在创建新连接前在连接池中查找空闲连接，查找失败时才会创建新的连接。

## 6.Nginx Lua实战

### 6.1 访问统计实战

得益于 Nginx 的高并发性能和 Redis 的高速缓存，基于 Nginx + Redis 的受访统计的架构设计比纯 Java 实现的架构设计在性能上高出很多。

```lua
local redisOp = require "luaScript.module.demo.redisOperator"
local red = redisOp:new()
red:open()
local visitCount = red:incrValue("demo:visitCount")
if visitCount then
    -- 设置 10s 过期
    red:expire("demo:visitCount", 10)
    -- 将结果存入 Nginx 变量 $count
    ngx.var.count = visitCount
end
red:close()
```

```nginx
location /visitCount {
    set $count 0;
    access_by_lua_file luaScript/module/demo/redisVisitCount.lua;
    echo "10s 內访问次数为: $count";
}
```

### 6.2 高并发访问实战

在有数据库连接的高并发场景下，我们往往会选择将数据放入 Redis 缓存中，Java API 会先访问 Redis 缓存，如果未命中则查询 DB，再将 Redis 中的结果进行更新或者删除。但是，常用的 java 容器（如 Tamcat、Jetty）的性能其实不是太高，QPS 往往会在 1000 以内。而Nginx 的性能是 Java 容器的 10 倍左右，并且稳定性更强，还不存在 FullGC 卡顿。

因此我们可以将 ”Java 容器 + Redis + DB“ 架构优化为 ”Nginx + Redis + Java 容器“，将 Java 容器的缓存判断、缓存查询前移到 Nginx，这样可以充分发挥 Nginx 的高并发优势和稳定性优势。

下面以秒杀系统的商品数据查询为例提供一个参考实现。

首先提供一个 Lua 操作缓存的类来实现：查询商品缓存、访问上游接口获取商品数据。

```lua
-- luaScrupt.module.demo.RedisCache

local RedisOp = require "luaScript.module.demo.RedisOperator"
local PREFIX = "GOOD_CACHE:"

--输出到日志文件
local function log(string)
    if  type(string)=="string" then
        ngx.log(ngx.DEBUG, string);
        return
    end
    if  type(string)=="table" then
        ngx.log(ngx.DEBUG, table.concat(string, " "));
        return
    end
    ngx.log(ngx.DEBUG, tostring(string));
end
--输出到日志文件
local function error(string)
    if  type(string)=="string" then
        ngx.log(ngx.ERR, string);
        return
    end
    if  type(string)=="table" then
        ngx.log(ngx.ERR, table.concat(string, " "));
        return
    end
    ngx.log(ngx.ERR, tostring(string));
end

local _RedisCacheDemo = {}
function _RedisCacheDemo:new()
    local object = setmetatable({}, _RedisCacheDemo)
    self.__index = self
    return object
end

-- 获取缓存数据
function _RedisCacheDemo:getCache(goodId)
    local red = RedisOp:new()
    if not red:open() then
        error("redis 连接失败")
        return nil;
    end
    local json = red:getValue(PREFIX .. goodId)
    red:close()
    if not json or json == ngx.null then
        log(goodId .. "的缓存没有命中")
        return nil
    end
    log(goodId .. "的缓存命中")
    return json
end

-- 通过子请求访问上游接口
function _RedisCacheDemo:goUpstream()
    local request_method = ngx.var.request_method
    local args = nil
    if "GET" == request_method then
        args = ngx.req.get_uri_args()
    elseif "POST" == request_method then
        ngx.req.read_body()
        args = ngx.req.get_post_args()
    end

    -- 访问上游接口
    local res = ngx.location.capture("/java/good/detail", {
            method = ngx.HTTP_GET,
            args = args
        }
    )
    log("数据获取成功: ", res.body)
    return res.body
end

return _RedisCacheDemo
```

然后在 Nginx 配置文件中定义两个接口：一个模拟 Java 容器的商品查询接口：/java/good/detail；另一个作为外部调用的商品查询接口：/good/detail。

```nginx
# 首先从缓存中查找商品，未命中时调用上游接口
location = /good/detail {
    content_by_lua_block {
        local goodId = ngx.var.arg_goodId
        -- 从缓存中查询
        local RedisCache = require "luaScript.module.demo.RedisCache"
        local redisCache = RedisCache:new()
        local json = redisCache:getCache(goodId)
        -- 判断缓存是否命中
        if not json then
            ngx.say("缓存没有命中，访问上游接口<br>")
            json = redisCache:goUpstream()
        else
            ngx.say("缓存已命中<br>")
        end
        ngx.say("商品信息: ", json)
    }
}
```

```nginx
# 模拟 Java 后台服务，查询商品并缓存
location = /java/good/detail {
    internal;
    content_by_lua_block {
        local PREFIX = "GOOD_CACHE:"
        local RedisOp = require "luaScript.module.demo.RedisOperator"
        local goodId = ngx.var.arg_goodId
        -- 从数据库查找数据，这里简化
        local json = '{goodId:' .. goodId .. ',goodname:商品名称}'
        -- 将商品缓存到 redis
        local red = RedisOp:new()
        red:open()
        red:setValue(PREFIX .. goodId, json)
        red:expire(PREFIX .. goodId, 60)
        red:close()
        ngx.say(json)
    }
}
```

访问 /good/detail 结果如下：

```bash
缓存没有命中，访问上游接口
商品信息: {goodId:3,goodname:商品名称}

缓存已命中
商品信息: {goodId:3,goodname:商品名称}
```

### 6.3 黑名单拦截实战

实现 IP 黑名单拦截有很多途径：

- 在操作系统层面配置 `iptables` 防火墙规则，拒绝黑名单中 IP 的网络请求
- 使用 Nginx 网关的 `deny` 配置指令拒绝黑名单中 IP 的网络请求
- 在 Nginx 网关的 `access` 处理阶段，通过 Lua 脚本检查客户端 IP 是否在黑名单中
- 在 Spring Cloud 内部网关的过滤器中检查客户端 IP 是否在黑名单中

以上检查方式都是基于一个静态的、提前准备好的黑名单进行。如果需要动态配置黑名单，则可以使用 Nginx 网关配合 Redis 实现。

首先是黑名单的组成，黑名单应该包括静态部分和动态部分。静态部分为系统管理员通过控制台设置的黑名单。动态部分主要通过流计算框架完成，具体方法为：将 Nginx 的访问日志通过 Kafka 消息中间件发送到流计算框架，然后通过滑动窗口机制计算出窗口内相同 IP 的访问计数，将超出阈值的 IP 动态加入黑名单中，流计算框架可以选用 Apache Flink 或者 Apache Storm。当然也可以使用 RxJava 滑动窗口进行访问计数的统计。

这里假设 IP 黑名单已经生成并且定期更新在 Redis 中。Nginx 网关可以直接从 Redis 获取计算好的 IP 黑名单，但是为了提高黑名单的读取速度，并不是每一次请求过滤都从 Redis 读取黑名单，而是从本地的共享内存中获取，同时定期将 Redis 的黑名单更新到本地共享内存。

<img src="img/第03章_Nginx Lua/image-20240113170601174.png" alt="image-20240113170601174" style="zoom:67%;" />

以下是一个 Lua 脚本实现：

```lua
-- luaScript/module/demo/Black_ip_filter.lua

local RedisOp = require "luaScript.module.demo.RedisOperator"

--输出到日志文件
local function log(string)
    if  type(string)=="string" then
        ngx.log(ngx.DEBUG, string);
        return
    end
    if  type(string)=="table" then
        ngx.log(ngx.DEBUG, table.concat(string, " "));
        return
    end
    ngx.log(ngx.DEBUG, tostring(string));
end
--输出到日志文件
local function error(string)
    if  type(string)=="string" then
        ngx.log(ngx.ERR, string);
        return
    end
    if  type(string)=="table" then
        ngx.log(ngx.ERR, table.concat(string, " "));
        return
    end
    ngx.log(ngx.ERR, tostring(string));
end

-- 获取客户端 IP
local function getClientIP()
    local clientIP = ngx.req.get_headers()["X-Real-IP"]
    if clientIP == nil then
        clientIP = ngx.req.get_headers()["X-Forwarded-For"]
    end
    if clientIP == nil then
        clientIP = ngx.var.remote_addr
    end
    return clientIP;
end

local ip = getClientIP()
-- 获取共享变量
local black_ip_list = ngx.shared.black_ip_list
local last_update_time = black_ip_list:get("last_update_time")
if last_update_time ~= nil then
    -- 如果本地缓存更新时间小于 60 秒则使用本地缓存进行判断
	local dif_time = ngx.now() - last_update_time
    if dif_time < 60 then
        if black_ip_list:get(ip) then
            return ngx.exit(ngx.HTTP_FORBIDDEN)
        end
        return
    end
end

-- 如果本地缓存过期，则同步 redis 到本地
local KEY = "limit:ip:blacklist"
local red = RedisOp:new()
red:open()
local ip_blacklist = red:getSmembers(KEY)
red:close()

if not ip_blacklist then
    log("black ip set is null")
 	return
else
    -- 将字典中的当前所有数据设置为过期
    black_ip_list:flush_all()
    
    -- 同步 redis 黑名单到本地缓存
    for i, ip in ipairs(ip_blacklist) do
        black_ip_list:set(ip, true)
    end
    black_ip_list:set("last_update_time", ngx.now())
end

-- 再次进行黑名单判断
if black_ip_list:get(ip) then
    return ngx.exit(ngx.HTTP_FORBIDDEN)
end
```

在 Nginx 配置文件中执行该脚本，另外由于 lua 脚本中使用了名为 black_ip_list 的共享内存进行黑名单本地缓存，因此需要在 http 上下文中配置共享内存空间：

```nginx
http {
    charset utf-8;
    lua_shared_dict black_ip_list 1m;
    keepalive_timeout 60;
    server {
        location /black_ip_demo {
            access_by_lua_file luaScript/module/demo/Black_ip_filter.lua;
            echo "恭喜，没有被拦截";
        }
    }
}
```

这里使用 `lua_shared_dict` 指令定义了一块 1MB 的共享内存。访问 /black_ip_demo 查看结果：

```bash
恭喜，没有被拦截
```

在 redis 服务器中添加 `Set` 类型的 key：`limit:ip:blacklist`，并加入当前客户端 IP，再次访问：

```bash
403 Forbidden
openresty/1.21.4.3
```

### 6.4 Nginx Lua共享内存

Nginx Lua 共享内存就是在内存块中分配出一个内存空间，该空间是一种字典结构，类似于 Java Map 的 Key-Value 映射结构。同一个 Nginx 下的 Worker 进程都能访问存储在这里面的数据。

Lua 中定义共享内存非常简单：

- 语法：`lua_shared_dict <DICT> <SIZE>`
- 上下文：`http` 配置块

```lua
lua_shared_dict black_ip_list 1m;
```

对于共享内存的引用可以使用以下两种方式：

- `ngx.shared.DICT`
- `ngx.shared["DICT"]`

常用 API 如下：

| API          | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| 取值         | `value, flags = ngx.shared.DICT:get(key)`                    |
| 设置值       | `success, err, forcible = ngx.shared.DICT:set(key, value, exptime?, flags?)`<br />可选参数 `exptime` 为过期时间，单位 s，如果不设置则默认永久；<br />可选参数 `flags` 表示额外的缓存内容，如果设置则可以通过 get 方法获取 |
| 删除数据项   | `ngx.shared.DICT:delete(key)`                                |
| 设置过期时间 | `ngx.shared.DICT:expire(key, exptime)`<br />`exptime` 为过期时间，单位 s |
| 查询过期时间 | `ttl, err = ngx.shared.DICT:ttl(key)`                        |
| 全部过期     | `ngx.shared.DICT:flush_all()`<br />将字典中所有数据项设置为过期，此操作并没有真正地删除数据 |
| 清除过期数据 | `flushed = ngx.shared.DICT:flush_expired(max_count?)`<br />清除字典中的过期数据项，可选参数 `max_count` 表示清楚数量，不设置则清除所有的过期数据 |

共享内存的 API 方法都是==原子操作==，`lua_shared_dict` 定义的同一个共享内存区域自带锁，可以避免多个 Worker 并发访问的问题。在新增数据时，如果字典的内存区域不够，`ngx.shared.DICT.set` 方法就会根据 LRU 算法淘汰一部分内容。当 nginx 退出时，共享内存中的数据项都会丢失。
