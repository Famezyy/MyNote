# 扩展：Nginx Lua指令

## 1.Nginx Lua内置方法

|               内置方法                | 说明                                                         |
| :-----------------------------------: | ------------------------------------------------------------ |
|            `ngx.say(...)`             | 输出相应内容到客户端，自动添加 `"\n"` 换行符                 |
|       `ngx.log(log_level, ...)`       | 按 `log_level` 的级别打印日志到 error.log 文件               |
|             `Print(...)`              | 输出到 error.log 日志文件，等价于 `ngx.log(ngx.NOTICE, ...)` |
|           `ngx.print(...)`            | 输出相应内容到客户端                                         |
|          `ngx.exit(status)`           | 如果 status >= 200，此方法就会结束当前请求处理，并且返回 `status` 状态到客户端；如果 status = 0，此方法就会结束请求处理的当前阶段，进入下一个请求处理阶段 |
|         `ngx.send_headers()`          | 显示地发送响应头。当调用 `ngx.say()/ngx.print()` 时，`ngx_lua` 模块会自动发送响应头，可以通过 `ngx.headers_sent` 内置变量的值判断是否发送了响应头 |
|        `ngx.exec(uri, args?)`         | 内部跳转到 URI 地址                                          |
|      `ngx.redirect(uri, args?)`       | 外部跳转到 URI 地址                                          |
| `ngx.location.capture(uri, options?)` | 发起一个子请求                                               |
|  `ngx.location.capture_multi(uris)`   | 发起多个子请求；参数 `uris` 是一个 `table`，它的格式为：`{ {uri,options?}, {uri,options?}, ... }` |
|         `ngx.is_subrequest()`         | 当前请求是不是子请求                                         |
|         `ngx.sleep(seconds)`          | 无阻塞地休眠 `seconds` 秒数                                  |
|           `ngx.get_phase()`           | 获取当前 Lua 脚本的执行阶段名称，为下列其中之一：<br />init、init_worker、ssl_cert、ssl_session_fetch、ssl_session_store、set、rewrite、balancer、access、content、header_filter、body_filter、log、timer |
|        `ngx.req.start_time()`         | 请求的开始时间                                               |
|       `ngx.req.http_version()`        | 请求的 HTTP 版本号                                           |
|        `ngx.req.raw_header()`         | 获取原始的请求头（包括请求行）                               |
|        `ngx.req.get_method()`         | 获取请求方法                                                 |
|     `ngx.req.set_method(method)`      | 覆盖当前请求的方法                                           |
|       `ngx.req.get_uri_args()`        | 获取请求参数                                                 |
|       `ngx.req.get_post_args()`       | 获取 post 请求内容体，用法和 `ngx.req.get_headers()` 类似，调用此方法前必须调用 `ngx.req.read_body()` 来读取 body |
|        `ngx.req.get_headers()`        | 获取请求头，默认只获取前 100 个；如果当前请求有多个请求头，则返回的是 `table`；如果想要获取全部请求头，可以使用 `ngx.req.get_headers(0)` |
|       `ngx.resp.get_headers()`        | 获取响应头，类似于 `ngx.req.get_headers()`                   |
|         `ngx.req.read_body()`         | 读取当前请求的请求体                                         |
|   `ngx.req.set_header(name, value)`   | 为当前请求设置一个请求头，如果指定的请求头名称存在则覆盖     |
|     `ngx.req.clear_header(name)`      | 为当前请求删除名称为 name 的请求头                           |
|     `ngx.req.set_body_data(data)`     | 设置当前请求的请求体为 data                                  |
|   `ngx.req.init_body(buffer_size?)`   | 为当前请求创建一个空的请求体。如果 `buffer_size` 不为空，则新请求体的大小为 `buffer_size`。如果 `buffer_size` 参数为空，那么新请求体的大小为 `client_body_buffer_size` 指令设置的请求体大小。如果未进行特定设置，默认的请求体大小为 8KB（32 位）或者 16KB（64 位） |
|         `ngx.escape_uri(str)`         | 对 uri 字符串进行编码                                        |
|        `ngx.unescape_uri(str)`        | 对编码过的 url 字符串进行解码                                |
|       `ngx.encode_args(table)`        | 将 Lua `table` 编码为一个参数字符串                          |
|        `ngx.decode_args(str)`         | 将参数字符串解码为一个 Lua `table`                           |
|       `ngx.encode_base64(str)`        | 将字符串 str 编码为 base 64 摘要                             |
|       `ngx.decode_base64(str)`        | 将 base 64 摘要解码成原始字符串                              |
|     `ngx.hmac_sha1(secret, str)`      | 将字符串 str 编码成二进制格式的 hmac_sha1 哈希摘要，并使用 `secret` 进行加密。该二进制摘要可使用 `ngx.encode_base64(digest)` 方法进一步将其编码成字符串 |
|            `ngx.md5(str)`             | 将字符串 str 编码成 16 进制 MD5 摘要                         |
|          `ngx.md5_bin(str)`           | 将字符串 str 编码成 2 进制 MD5 摘要                          |
|       `ngx.quote_sql_str(str)`        | SQL 语句转义，按照 MySQL 的格式进行转义                      |
|             `ngx.today()`             | 获取当前日期                                                 |
|             `ngx.time()`              | 获取 UNIX 时间戳                                             |
|              `ngx.now()`              | 获取当前时间                                                 |
|          `ngx.update_time()`          | 刷新时间后再返回                                             |
|           `ngx.localtime()`           | 获取 yyyy-mm-dd hh:mm:ss 格式的本地时间                      |
|          `ngx.cookie_time()`          | 获取可用于 cookie 值的时间                                   |
|           `ngx.http_time()`           | 获取可用于 HTTP 头的时间                                     |
|        `ngx.parse_http_time()`        | 解析 HTTP 头的时间                                           |
|     `ngx.config.nginx_version()`      | 获取 Nginx 版本号                                            |
|    `ngx.config.ngx_lua_version()`     | 获取 `ngx_lua` 模块的版本号                                  |

## 2.Nginx Lua内置变量

|   内置变量   | 说明                                                         |
| :----------: | ------------------------------------------------------------ |
|  `ngx.arg`   | 类型为 `table`，`ngx.arg.VARIABLE` 用于获取 `ngx_lua`配置指令后面的调用参数值。例如可以获取跟在 `set_by_lua` 指令后面的调用参数值 |
|  `ngx.var`   | 类型为 `table`，`ngx.var.VARIABLE` 引用某个 Nginx 变量。如果需要对 Nginx 变量进行赋值，如 `ngx.var.b = 2`，则变量 b 必须提前声明；另外可以使用 `ngx.var[捕获组序号]` 的格式引用 `location` 配置块中被正则表达式捕获的捕获组 |
|  `ngx.ctx`   | 类型为 `table`，可以用来访问当前请求的 Lua 上下文数据，生存周期与当前请求相同 |
| `ngx.header` | 类型为 `table`，用于访问 HTTP 响应头，可以通过 `ngx.header.HEADER` 形式引用某个头，比如可以通过 `ngx.header.set_cookie` 访问响应头部的 Cookie 信息，同样是 `table` 类型 |
| `ngx.status` | 用于设置当前请求的 HTTP 响应码                               |
|  `ngx.req`   | 获取请求相关参数，例如 `ngx.req.get_url_args()` 可以获取 query 参数 |

## 3.Nginx Lua内置常量

| 常量类型        | 说明                                                         |
| :-------------- | :----------------------------------------------------------- |
| 核心常量        | ngx.OK (0)<br />ngx.ERROR (-1)<br />ngx.AGAIN (-2)<br />ngx.DONE (-4)<br />ngx.DECLINED (-5)<br />ngx.null |
| HTTP 方法常量   | ngx.HTTP_GET<br />ngx.HTTP_HEAD<br />ngx.HTTP_PUT<br />ngx.HTTP_POST<br />ngx.HTTP_DELETE<br />ngx.HTTP_OPTIONS<br />ngx.HTTP_MKCOL<br />ngx.HTTP_COPY<br />ngx.HTTP_MOVE<br />ngx.HTTP_PROPFIND<br />ngx.HTTP_PROPPATCH<br />ngx.HTTP_LOCK<br />ngx.HTTP_UNLOCK<br />ngx.HTTP_PATCH<br />ngx.HTTP_TRACE |
| HTTP 状态码常量 | ngx.HTTP_OK (200)<br />ngx.HTTP_CREATED (201)<br />ngx.HTTP_SPECIAL_RESPONSE (300)<br />ngx.HTTP_MOVED_PERMANENTLY (301)<br />ngx.HTTP_MOVED_TEMPORARILY (302)<br />ngx.HTTP_SEE_OTHER (303)<br />ngx.HTTP_NOT_MODIFIED (304)<br />ngx.HTTP_BAD_REQUEST (400)<br />ngx.HTTP_UNAUTHORIZED (401)<br />ngx.HTTP_FORBIDDEN (403)<br />ngx.HTTP_NOT_FOUND (404)<br />ngx.HTTP_NOT_ALLOWED (405)<br />ngx.HTTP_GONE (410)<br />ngx.HTTP_INTERNAL_SERVER_ERROR (500)<br />ngx.HTTP_METHOD_NOT_IMPLEMENTED (501)<br />ngx.HTTP_SERVICE_UNAVAILABLE (503)<br />ngx.HTTP_GATEWAY_TIMEOUT (504) |
| 日志类型常量    | ngx.STDERR<br />ngx.EMERG<br />ngx.ALERT<br />ngx.CRIT<br />ngx.ERR<br />ngx.WARN<br />ngx.NOTICE<br />ngx.INFO<br />ngx.DEBUG |
