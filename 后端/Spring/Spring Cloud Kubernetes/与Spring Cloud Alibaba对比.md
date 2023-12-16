# 特点

## 1.对比Spring Cloud Alibaba

|功能|Spring Cloud Alibaba|描述|
|--|--|--|
|服务发现|Nacos|Kubernetes 基于探针自带服务发现|
|配置管理|Nacos|利用 Kubernetes 的 ConfigMap 和 Secret|
|负载均衡|Spring Cloud LoadBalancer|基于 Service 负载均衡|
|API 网关|Spring Cloud Gateway|Nginx Ingress Controller|
|自动扩缩容|NA|HPA|
|滚动更新|NA|自带滚动更新、灰度发布|
|任务调度|NA|Kubernetes Job|
|流控|Alibaba Sentinel|NA，可以扩展Istio|
|认证授权|Spring Security & Oauth2|集成 RBAC 的安全机制，如身份验证、授权和加密通信|
|监控与日志|Nacos、Sentinel 自带简陋 Dashboard|Kubernetes 集群管理（K8S Dashboard、Kuboard、Rancher）|
|厂商依赖|弱绑定阿里云|适用于所有 K8S 标准环境|
|分布式事务|Alibaba Seata|NA|
|构建复杂度|复杂|开箱即用|
|开发生产一致性|不一致|开发测试调试都在 K8S 上，生产部署时只需要修改 namespace|

## 2.面临的课题

- Kubernetes 是一个复杂的系统，学习成本大
- 应用必须运行在 K8S 中，本地调试困难，常见的方案是使用 KT-Connect、Telepresence 等本地代理工具接入远程 K8S 环境
- K8S 集群需要消耗大量计算资源
- 需要更多的配置：网络配置、存储配置、服务发现、负载均衡
- 运维复杂：监控、日志管理、故障排查。扩展
- Spring Cloud Kebernetes 社区较小