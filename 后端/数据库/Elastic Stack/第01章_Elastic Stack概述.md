# 第01章_Elastic Stack概述

## 1.Elastick Stack简介

Elastic Stack 包括 ElasticSearch、Kibana、Beats 和 Logstash（也称为 ELK Stack）。

- **ElasticSearch**

  一个开源的高扩展的分布式全文搜索引擎，是整个 Elastic Stack 技术栈的核心，可以几乎实时的存储、检索数据；本身扩展性好，可以扩展到上百台服务器，处理 PB 级别的数据。

- **Kibana**

  一个免费开源的用户界面，能够对 ElasticSearch 数据进行可视化，并在 Elastic Stack 中进行导航。

- **Beats**

  免费开源平台，集合了多种单一用途数据采集器，可以从成百上千万台机器和系统向 Logstash 或 ElasticSearch 发送数据。

- **Logstash**

  免费开源的服务器端数据处理管道，能够从多个来源采集数据，转换数据，然后将数据发送到存储中心。

使用 Elastic Stack 可以收集的日志如下：

- 容器管理编排工具：docker、K8S
- 负载均衡服务器：lva、haproxy、nginx
- web 服务器：httpd、nginx、tomcat
- 数据库：mysql、redis 等
- 存储：nfs、Ceph 等
- 系统：message、security 等

## 2.常见架构

### 2.1 EFK

```bash
源数据层 (nginx,tomcat) -> 数据采集层 (Filebeat) -> 数据存储层 (ElasticSearch)
```

### 2.2 ELK

```bash
源数据层 (nginx,tomcat) -> 数据采集/转换层 (Logstash) -> 数据存储层 (ElasticSearch)
```

### 2.3 ELFK

```bash
源数据层 (nginx,tomcat) -> 数据采集 (Filebeat) -> 转换层 (Logstash) -> 数据存储层 (ElasticSearch)
```

### 2.4 ELFK + Kafka

在 ELFK 架构中，由于 Logstash 是流量瓶颈，为了防止其崩溃，在中间加一层消息中间件作为缓存。

```bash
源数据层 (nginx,tomcat) -> 数据采集 (Filebeat) -> 数据缓存层 (Kafka) -> 转换层 (Logstash) -> 数据存储层 (ElasticSearch)
```

## 3.部署

### 3.1 环境准备

#### 1.准备虚拟机

准备三台虚拟机，CPU 2 核 内存 4 G 用作集群节点，并配置好域名解析。

#### 2.修改数据源

#### 3.修改sshd服务优化

#### 3.关闭防火墙

#### 4.禁用selinux



#### 5.集群时间同步



