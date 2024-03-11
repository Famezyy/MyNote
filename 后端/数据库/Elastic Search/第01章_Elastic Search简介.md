# 第01章_Elastic Search简介

## 1.官方文档介绍

- **[官网地址](http://elastic.co/)**
- **[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)**
- **[官方下载](https://www.elastic.co/cn/downloads/past-releases#elasticsearch)**
- **[中文社区](https://www.elastic.org.cn/archives/chapter1)**

直到 ES 8.0 之前，JDK 1.8 支持所有的 ES 版本，但是在 ES 8.0 之后，对于 ES 8.0 而言，JDK版本只有一个选择，即 JDK 17；对于ES 8.1 及以上版本而言，支持 JDK 17、JDK 18。

从 7.x 开始以后的版本 ES 均自带 JDK，所以即使不安装 JDK 也可正常运行 ES。

- **[JDK 兼容性](https://www.elastic.co/cn/support/matrix#matrix_jvm)**

- **[操作系统兼容性](https://www.elastic.co/cn/support/matrix)**

- **[自身兼容性](https://www.elastic.co/cn/support/matrix#matrix_compatibility)**

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

## 3.服务的安装和启动

运行 `bin/elasticsearch` 可执行文件即可启动，启动后访问 `localhost:9200` 即可访问。

> **注意**
>
> - 在 Windows 环境变量的 classpath 中如果配置了 dt.jar 和 tools.jar、logkit-2.0.jar 路径，则需要删除该变量，否则会在启动时报错
> - ES 8 默认启动 `Security` 功能，在第一次启动后会在配置文件 `config/elasticsearch.yml` 中增加几行配置，需要手动将 `xpack.security.enabled` 修改为 false
> - 解决 Windows 下中文乱码问题：打开 `conf/jvm.options` 添加 `-Dfile.encoding=GBK`
> - 如果 9200 无法访问，可能是发生了端口占用，可以查看控制台上监听的端口，或修改配置文件指定 `http.port`

