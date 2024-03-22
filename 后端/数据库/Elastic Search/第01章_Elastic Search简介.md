# 第01章_Elastic Search简介

## 1.基础简介

`Elasticsearch`（后称为 ES）是一个天生支持**分布式**的，基于 Json 的搜索、聚合分析和存储引擎。

- **[官网地址](http://elastic.co/)**
- **[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)**
- **[官方下载](https://www.elastic.co/cn/downloads/past-releases#elasticsearch)**
- **[中文社区](https://www.elastic.org.cn/categories/docs)**

直到 ES 8.0 之前，JDK 1.8 支持所有的 ES 版本，但是在 ES 8.0 之后，对于 ES 8.0 而言，JDK版本只有一个选择，即 JDK 17；对于ES 8.1 及以上版本而言，支持 JDK 17、JDK 18。

从 7.x 开始以后的版本 ES 均自带 JDK，所以即使不安装 JDK 也可正常运行 ES。

- **[JDK 兼容性](https://www.elastic.co/cn/support/matrix#matrix_jvm)**

- **[操作系统兼容性](https://www.elastic.co/cn/support/matrix)**

- **[自身兼容性](https://www.elastic.co/cn/support/matrix#matrix_compatibility)**

### 1.1 特点

- **开源、免费**：基于 Apache License 2.0 开源协议，并且完全免费
- **基于Java语言**：Elasticsearch 基于 Java 语言开发，运行在 JVM 环境中
- **基于Lucene框架：**基于开源的 Apache Lucene 框架开发，扩展了丰富的功能
- **原生分布式**：不依赖与 Zookeeper，自带分布式解决方案
- **高性能**：支持海量数据的全文检索。支持 PB 级数据**秒内响应**
- **可伸缩**：弹性搜索，可根据不同规模服务对性能需要的不同而动态扩展或收缩性能
- **易扩展**：支持非常方便的横向扩展集群
- **开箱即用**：对于小公司，解压后**零配置**启动服务即可立即使用，门槛极低
- **跨编程语言**：支持 Java、Golang、Python、C#、PHP 等多种变成语言，几乎所有语言开发者都可以使用 Elasticsearch

### 1.2 应用场景

只要是用到搜索的场景，ES 几乎都可以说是最好的选择。

ES 支持的搜索类型：

- 结构化搜索：结构化数据通常可以通过查询和分析工具进行处理和分析，从中提取有用的信息和洞察
- 非结构搜索：如文本、图像、音频和视频等，它们没有明确的结构和格式，处理和分析起来更加困难
- 文本搜索
- 地理位置搜索

### 1.3 性能

#### 1.擅长场景

ES 最擅长从海量数据中检索少量相关数据，但不擅长单次查询大量数据（大单页）。

#### 2.写入实时性

ES 是 OLAP 系统，侧重于海量数据的检索，而写入实时性并不是很高，默认 1 秒，也就是 ES 缓冲区 Buffer 的刷新间隔时间。ES 并非忽略了对写入性能的优化，而是“有意为之”，其原因就在于基于 ES 的写入机制，其写入实时性和大数据检索性能是一个二选一的行为。实际上生产环境中我们经常通过“**牺牲写入实时性**”的操作来换取更高更快的“数据检索”性能。

#### 3.不支持事务

正因为 ES 的写入实时性并不高，如果我们需要快速响应用户请求，我们常采取的手段就是使用缓存，但是在很多高并发的场景下，我们需要数据保持强一致性（如银行系统），因此需要使用具有 ACID 特性的数据库来支持，而 MySQL 就是一个比较好的选择。

#### 4.极限性能

PB（1PB = 1024TB = 1024²GB）级数据秒内响应。

### 1.4 选型

|                  | Elasticsearch                  | Solr                                                      | MongoDB                    | MySQL                |
| :--------------- | :----------------------------- | :-------------------------------------------------------- | :------------------------- | :------------------- |
| DB类型           | 搜索引擎                       | 搜索引擎                                                  | 文档数据库                 | 关系型数据库         |
| 基于何种框架开发 | Lucene                         | Lucene                                                    |                            |                      |
| 基于何种开发语言 | Java                           | Java                                                      | C++                        | C、C++               |
| 数据结构         | FST、Hash 等                   |                                                           |                            | B+ Trees             |
| 数据格式         | Json                           | Json/XML/CSV                                              | Json                       | Row                  |
| 分布式支持       | 原生支持                       | 支持                                                      | 原生支持                   | 不支持               |
| 数据分区方案     | 分片                           | 分片                                                      | 分片                       | 分库分表             |
| 业务系统类型     | OLAP                           | OLAP                                                      | OLTP                       | OLTP                 |
| 事务支持         | 不支持                         | 不支持                                                    | 多文档 ACID 事务           | 支持                 |
| 数据量级         | PB 级                          | TB 级 ~ PB 级                                             | PB 级                      | 单库 3000 万         |
| 一致性策略       | 最终一致性                     | 最终一致性                                                | 最终一致性、即时一致性     | 即时一致性           |
| 擅长领域         | 海量数据全文检索大数据聚合分析 | 大数据全文检索                                            | 海量数据 CRUD              | 强一致性 ACID 事务   |
| 劣势             | 不支持事务写入实时性低         | 海量数据的性能不如 ES 随着数据量的不断增大，稳定性低于 ES | 弱事务支持不支持 join 查询 | 大数据全文搜索性能低 |
| 查询性能         | ★★★★★                          | ★★★★                                                      | ★★★★★                      | ★★★                  |
| 写入性能         | ★★                             | ★★                                                        | ★★★★                       | ★★★                  |

## 2.配置简介

### 2.1 目录结构

| **目录名称** | **描述**                                                     |
| :----------- | :----------------------------------------------------------- |
| bin          | 可执行脚本文件，包括启动 elasticsearch 服务、插件管理、函数命令等。 |
| config       | 配置文件目录，如 elasticsearch 配置、角色配置、jvm 配置等。  |
| lib          | elasticsearch 所依赖的 java 库。                             |
| data         | 默认的数据存放目录，包含节点、分片、索引、文档的所有数据，**生产环境要求必须修改**。 |
| logs         | 默认的日志文件存储路径，**生产环境务必修改**。               |
| modules      | 包含所有的 Elasticsearch 模块，如 Cluster、Discovery、Indices 等。 |
| plugins      | 已经安装的插件的目录。                                       |
| jdk/jdk.app  | 7.x 以后特有，自带的 java 环境，8.x 版本自带 jdk 17          |

### 2.2 基础配置

配置文件放在 `config` 目录下的 `elasticsearch.yml`。

- `cluster.name`：集群名称，节点根据集群名称确定是否是同一个集群。默认名称为 elasticsearch，但应将其更改为描述集群用途的适当名称。不要在不同的环境中重用相同的集群名称，否则节点可能会加入错误的集群
- `node.name`：节点名称，集群内唯一，默认为主机名，但可以在配置文件中显式配置
- `path.data`：存放 data 的目录，默认保存在 ES 的根目录
- `path.log`：存放 log 的目录，默认保存在 ES 的根目录
- `network.host`： 节点对外提供服务的地址以及集群内通信的 IP 地址，例如127.0.0.1和 [::1]
- `http.port`：对外提供服务的端口号，默认 9200
- `transport.port`：节点通信端口号，默认 9300

### 2.3 开发模式和生产模式

**（1）开发模式**

开发模式是默认配置（未配置集群发现设置），如果用户只是出于学习目的，而引导检查会把很多用户挡在门外，所以 ES 提供了一个设置项 `discovery.type=single-node`。此项配置为指定节点为单节点发现以绕过引导检查。

**（2）生产模式**

当用户修改了有关集群的相关配置会触发生产模式，在生产模式下，服务启动会触发ES的引导检查或者叫启动检查（bootstrap checks），所谓引导检查就是在服务启动之前对一些重要的配置项进行检查，检查其配置值是否是合理的。引导检查包括对 JVM 大小、内存锁、虚拟内存、最大线程数、集群发现相关配置等相关的检查，如果某一项或者几项的配置不合理，ES 会拒绝启动服务，并且在开发模式下的某些警告信息会升级成错误信息输出。

引导检查十分严格，之所以宁可拒绝服务也要阻止用户启动服务，是为了防止用户在对 ES 的基本使用不了解的前提下启动服务而导致的后期性能问题无法解决或者解决起来很麻烦。因为一旦服务以某种不合理的配置启动，时间久了之后可能会产生较大的性能问题，但此时集群已经变得难以维护和扩展，ES 为了避免这种情况而做出了引导检查的设置，本来在开发模式下为警告的启动日志会升级为报错（Error）。

## 3.服务的安装和启动（开启security）

安装路径==不要有**空格**和**中文**==！

### 3.1 创建ES服务账号

在 Linux 环境下 ES 不允许使用 `root` 账号启动服务，如果你当前账号是 `root`，则需要创建一个专有账户（以下命令均在 `root` 账户下执行）。如果你的账号不是 `root` 账号，此步骤可以跳过。

创建账号步骤：

```bash
useradd elastic
passwd elastic
chown -R elastic:elastic {{espath}}
```

### 3.2 单节点集群

#### 1.启动命令

|              | **Windows**                                       | **MacOS**                                        | **Linux**                                   |
| :----------- | :------------------------------------------------ | :----------------------------------------------- | :------------------------------------------ |
| **命令行**   | `cd elasticsearch\bin`<br /> `.\elasticsearch -d` | `cd elasticsearch/bin`<br />`./elasticsearch -d` | `cd elasticsearch/bin # ./elasticsearch -d` |
| **图形界面** | 在 bin 目录下双击 elasticsearch.bat               | 在 bin 目录下双击 elasticsearch                  | —                                           |
| **Shell**    | `start \bin\elasticsearch.bat`                    | `open bin/elasticsearch`                         | —                                           |

#### 2.启动日志

ES在 7.x 版本时，控制台输出 started 时代表服务启动成功。和 7.x 版本不同，ES 8.x **首次**启动之后会输出以下信息，此时服务已经启动成功了：

```bash
-> Elasticsearch security features have been automatically configured!
-> Authentication is enabled and cluster connections are encrypted.

->  Password for the elastic user (reset with `bin/elasticsearch-reset-password -u elastic`):
  # 默认账号 elastic 的密码
  KylaVSTe6*lEbB1rNFTp

->  HTTP CA certificate SHA-256 fingerprint:
  # CA 访问密钥
  6c4d9d215d1ed1b7378a5ed5a6bff29f790068ebeb2d3670bb4722d40e7638ad

->  Configure Kibana to use this cluster:
* Run Kibana and click the configuration link in the terminal when Kibana starts.
* Copy the following enrollment token and paste it into Kibana in your browser (valid for the next 30 minutes):
 # kibana 访问 token
 eyJ2ZXIiOiI4LjEyLjAiLCJhZHIiOlsiMTcyLjIxLjE5Mi4xOjkyMDAiXSwiZmdyIjoiNmM0ZDlkMjE1ZDFlZDFiNzM3OGE1ZWQ1YTZiZmYyOWY3OTAwNjhlYmViMmQzNjcwYmI0NzIyZDQwZTc2MzhhZCIsImtleSI6IkhGUjlNSTRCczgxaHYwTzJva0lZOm1aRU9UbG16UVQ2NVdIZXpfLTN4dWcifQ==

# 配置其他 node 加入集群
->  Configure other nodes to join this cluster:
* On this node:
  - Create an enrollment token with `bin/elasticsearch-create-enrollment-token -s node`.
  - Uncomment the transport.host setting at the end of config/elasticsearch.yml.
  - Restart Elasticsearch.
* On other nodes:
  - Start Elasticsearch with `bin/elasticsearch --enrollment-token <token>`, using the enrollment token that you generated.
```

首次启动 Elasticsearch 后，会自动进行以下安全配置：

- 为传输层和 HTTP 层生成 TLS [证书和密钥](https://www.elastic.co/guide/en/elasticsearch/reference/current/configuring-stack-security.html#stack-security-certificates)
- TLS 配置设置被写入 elasticsearch.yml
- 为 elastic 用户生成密码
- 为 Kibana 生成一个注册令牌

#### 3.修改账号密码

在 ES 8.x 版本以后，elasticsearch-setup-passwords 设置密码的工具已经被弃用删除，此命令为 7.x 之前第一次生成密码时使用，8.x 在第一次启动的时候会自动生密码。

如果需要修改账户密码，需进行以下操作：

```bash
bin/elasticsearch-reset-password

[-a, --auto] [-b, --batch] [-E {KeyValuePair}]
[-f, --force] [-h, --help] [-i, --interactive]
[-s, --silent] [-u, --username] [--url] [-v, --verbose]
```

使用此命令重置本地领域中的任何用户或任何内置用户的密码。默认情况下，系统会为您生成一个强密码。要显式设置密码，请使用以交互模式运行该工具 `-i`。该命令在 [文件领域](https://www.elastic.co/guide/en/elasticsearch/reference/current/file-realm.html)中生成（并随后删除）一个临时用户，以运行更改用户密码的请求。

- `-a, --auto`：将指定用户的密码重置为自动生成的强密码（默认）

- `-b, --batch`：运行重置密码过程而不提示用户进行验证

- `-E {KeyValuePair}`：配置标准 Elasticsearch 或 X-Pack 设置

- `-f, --force`：强制命令针对不健康的集群运行

- `-h, --help`：返回所有命令参数

- `-i, --interactive`：提示输入指定用户的密码。使用此选项显式设置密码

- `-s --silent`：在控制台中显示最小输出

- `-u, --username`：本机领域用户或内置用户的用户名
- `–url`：指定工具用于向 Elasticsearch 提交 API 请求的基本 URL（本地节点的主机名和端口）。默认值由 elasticsearch.yml 文件中的设置确定；如果 `xpack.security.http.ssl.enabled` 设置为 true，则必须指定 HTTPS URL

- `-v --verbose`：在控制台中显示详细输出

**比如**：

为 `elastic` 账号自动生成新密码，输出至控制台：

```bash
bin/elasticsearch-reset-password -u elastic
```

手工指定 `user1` 的新密码：

```bash
bin/elasticsearch-reset-password --username elastic -i
```

指定服务地址和账户名：

```bash
bin/elasticsearch-reset-password --url "https://172.0.0.3:9200" --username elastic -i
```

### 3.3 验证服务状态

由于默认开启了 `security` 功能，需要通过 https://localhost:9200/ 访问，并要求输入用户名和密码。

> **注意**
>
> - 在 Windows 环境变量的 classpath 中如果配置了 dt.jar 和 tools.jar、logkit-2.0.jar 路径，则需要删除该变量，否则会在启动时报错
> - 解决 Windows 下中文乱码问题：打开 `conf/jvm.options` 添加 `-Dfile.encoding=GBK`
> - 如果 9200 无法访问，可能是发生了端口占用，可以查看控制台上监听的端口，或修改配置文件指定 `http.port`

### 3.4 构建本地集群

1. 新开启一个命令窗口，执行 `bin/elasticsearch-create-enrollment-token -s node`
2. 记录下生成的 token
3. 执行 `bin/elasticsearch --enrollment-token <token>` 加入集群
4. 访问 `localhost:9200/_cat/nodes?v` 查看节点状态

### 3.5 整合Kibana

1. 从官网下载 Kibana
2. 解压后执行 `bin/kibana`
3. 访问命令行打印的端口，粘贴 ES 生成的 Kibana 证书

生成新的 Kibana 证书：`elasticsearch-create-enrollment-token -s kibana --url "https://127.0.0.1:9200"`

## 4.服务的安装和启动（关闭security）

### 4.1 关闭Security

打开 Config 目录，修改 elasticsearch.yml 文件，添加以下配置：

```bash
xpack.security.enabled: false
```

### 4.2 启动单机服务

启动后不会打印 security 和 token 相关信息，直接访问 `http://127.0.0.1:9200` 或者 `http://localhost:9200` 即可。

### 4.3 部署本地集群

部署集群时无需加入 token，直接本地启动后会自动识别节点并加入集群。

#### 1.单项目多节点启动

| 操作系统     | 命令                                                         |
| :----------- | :----------------------------------------------------------- |
| Linux、MacOS | **节点1：**`elasticsearch -Epath.data=../node1/data1 -Epath.logs=../node1/log1 -Enode.name=node1 -Ecluster.name=elastic.org.cn`<br />**节点2：**`elasticsearch -Epath.data=../node2/data2 -Epath.logs=../node2/log2 -Enode.name=node2 -Ecluster.name=elastic.org.cn`<br />**节点N：… …** |
| Windows      | **节点1：**`elasticsearch.bat -Epath.data=../node1/data1 -Epath.logs=../node1/log1 -Enode.name=node1 -Ecluster.name=elastic.org.cn`<br />**节点2**：`elasticsearch.bat -Epath.data=../node2/data2 -Epath.logs=../node2/log2 -Enode.name=node2 -Ecluster.name=elastic.org.cn`<br />**节点N**：… … |

> **注意**
>
> 指定的 `path` 路径是以当前执行路径开始的，因此上面的执行

#### 2.多项目多节点启动

| 操作系统 | 脚本                                                         |
| :------- | :----------------------------------------------------------- |
| MacOS    | `open /node1/bin/elasticsearch`<br />`open /node2/bin/elasticsearch`<br />`open /node3/bin/elasticsearch` |
| windows  | `start D:\node1\bin\elasticsearch.bat`<br />`start D:\node2\bin\elasticsearch.bat`<br />`start D:\node3\bin\elasticsearch.bat` |

## 5.浏览器插件

| 插件名称            | 功能介绍                                                   | 下载地址                                                     |
| :------------------ | :--------------------------------------------------------- | :----------------------------------------------------------- |
| Elasticsearch Head  | 方便查看集群节点数据方便管理和索引、分片支持同时连接多集群 | [Chrome下载](https://chrome.google.com/webstore/detail/multi-elasticsearch-head/cpmmilfkofbeimbmgiclohpodggeheim) [Github下载](https://github.com/mobz/elasticsearch-head) [安装教程](http://www.elastic.org.cn/archives/es-head) |
| Elasticsearch Tools | 方便查看节点资源占用可执行查询语句                         | [Chrome下载](https://chrome.google.com/webstore/detail/elasticsearch-tools/aombbfhbleaidjmbahldfbajjmgkgojl) |
| Elasticvue          | 功能强大对国人友好                                         | [Chrome下载](https://chrome.google.com/webstore/detail/elasticvue/hkedbapjpblbodpgbajblpnlpenaebaa)[Edge下载](https://microsoftedge.microsoft.com/addons/search/elasticvue?hl=zh-CN) |
