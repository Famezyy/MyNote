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

|    类型    |    名称    | 说明                                                         |
| :--------: | :--------: | ------------------------------------------------------------ |
|  `number`  |    实数    | 可以是整数、浮点数                                           |
|  `string`  |   字符串   | 字符串类型，值是不可改变的                                   |
| `boolean`  |  布尔类型  | false 和 `nil` 是假，其他为真                                |
|  `table`   | 数组、容器 | `table` 类型实现了一种抽象的“关联数组”，相当于 Java 中的 `Map` |
| `userdata` |     类     | 其他语言中的对象类型，转换过来就变成 `userdata` 类型，比如 Redis 返回的空值有可能就是 `userdata` 类型，判空的时候要注意！ |
|  `thread`  |    线程    | 和 Java 中的线程差不多，代表一条执行序列，拥有自己独立的栈、局部变量和命令指针 |
| `function` |    函数    | 由 C 或 Lua 编写的函数，属于一种数据类型                     |
|   `nil`    |   空类型   | 表示变量没被赋值，`nil` 类型就 `nil` 一个值，变量赋值成 `nil` 也表示删除变量 |

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

- 普通 Lua 数组的数字索引对应于 Java 的元素下标，是从 1 开始计数的。

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

Lua 定义了一个负责数组和容器操作的 `table` 模块，主要的字符串操作大致如下：

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

### 2.8 Lua的面向对象

在 Lua 中使用表（`table`）实现面向对象，一个表就是一个对象。表可以拥有所有 8 大数据类型的成员属性。

下面在 DataType.lua 模块中定义带有一个成员的 `_Square` 类。

```lua
-- 正方形类
_Square = {
    side = 0
}
_Square.__index = _Square

-- 类的方法 getArea
function _Square.getArea(self)
    return self.side * self.side;
end

-- 类的方法 new
function _Square.new(self, size)
    local cls = {}
    setmetatable(cls, self)
    cls.size = size or 0
    return cls;
end

-- 统一的模块对象
local _Module = {
    ...
    Square = _Square
}
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

所以在上面的代码中，`_Square` 表设置了 `_index` 属性的值为自身，当为新创建的 `new` 对象查找 `getArea` 方法时，需要在元表 `_Square` 表的 `__index` 属性中查找。

下面是一个继承的代码示例：

```lua
Rectangle = {
    length = 0,
    width = 0
}

Rectangle.__index = Rectangle

function Rectangle:new(length, width)
    local obj = setmetatable({}, Rectangle)
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

Square.__index = Square

function Square:new(size)
    local obj = Rectangle:new(size, size)
    obj.size = size or 0
    return obj
end

square = Square:new(20)
ngx.say(square:getArea())
ngx.say("<bar>")
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
|  `content_by_lua_block`   | 与 `content_by_lua` 指令类似，不同之处在于该指令直接在一堆花括号 `"{}"` 中编写 Lua 脚本源码，而不是在 Nginx 字符串中（需要特殊字符转义） |
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



#### 5.set_by_lua

#### 6.access_by_lua

#### 7.content_by_lua

### 3.3 内置常量和变量

### 3.4 实践

## 4.重定向与内部子请求

## 5.Nginx Lua操作Redis

## 6.实战