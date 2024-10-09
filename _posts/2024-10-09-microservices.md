---
typora-root-url: ../../../typora
---

[TOC]

# 服务架构演进

<img src="/images/spring_cloud/QQ20190710-125048@2x.jpg" alt="服务架构演进图" style="zoom:33%;" />



服务架构演进过程：抽取各个模块独立维护，独立部署的过程。

初创公司2 ~ 3个研发人员，ALL IN ONE 的框架开发效率最高。随着队伍的壮大，产品，用户，商品成立独立小组，拆出相应的模块，独立维护，相互不影响，根据不同流量，分配不同的服务器。然而也带来了问题：业务之间的相互调用问题。于是不同部门将一些公共服务，抽取出来，做成rpc 服务，供其他兄弟部门调用。



## ALL IN ONE

<img src="/images/spring_cloud/QQ20190710-125315@2x.jpg" alt="ALL IN ONE" style="zoom:33%;" />

关键点：数据访问层（ORM）是关键。

优点：

1. 简单。
2. 减少部署节点和成本。

缺点：

1. 扩展不容易。
2. 不易系统开发。
3. 商品的bug，可能影响到用户服务的稳定性

## 垂直应用架构

<img src="/images/spring_cloud/QQ20190710-125705@2x.jpg" alt="垂直应用架构" style="zoom:33%;" />

关键点：前端页面开发的 Web 框架（MVC）是关键。

每个独立应用：页面，http 服务，数据库。

独立扩展：用户量和商品访问量大了，增加对应的服务器就可以了。

独立开发，互补干扰。

性能扩展方便。

缺点：

1. 无法做到页面和业务逻辑的实现分离。页面改动频繁，业务逻辑修改较少。
2. 业务逻辑无法完全独立，大量应用需要交互。订单系统：需要用户和商品的服务。支付也需要用户的信息等等。



## 分布式服务架构

<img src="/images/spring_cloud/QQ20190710-125755@2x.jpg" alt="分布式服务框架" style="zoom:33%;" />

当垂直应用越来越多,应用之间交互不可避免,将核心业务抽取出来,作为独立的服
务,逐渐形成稳定的服务中心,使前端应用能更快速的响应多变的市场需求。此时,
用于提高业务复用及整合的分布式服务框架(RPC)是关键。

抽取出：Web 页面 和 rpc 服务。

因为web 和 业务服务不在同一台服务器。需要远程调用，所以上rpc（远程过程调用）。



# 注册中心

<img src="/images/spring_cloud/QQ20190710-125858@2x.jpg" alt="注册中心" style="zoom:33%;" />

通过注册中心：治理服务。

服务发现，服务动态扩容。



## 流动计算架构

<img src="/images/spring_cloud/QQ20190710-125947@2x.jpg" style="zoom:33%;" />



https://martinfowler.com/articles/microservices.html

# 微服务

优点：

1. 每个服务足够内聚，足够小



微服务技术栈：

| 微服务条目                               | 落地技术                                                     | 备注 |
| ---------------------------------------- | ------------------------------------------------------------ | ---- |
| 服务开发                                 | SpringBoot，Spring，SpringMVC                                |      |
| 服务配置与管理                           | Neffix 的 Archaius，阿里的Diamond                            |      |
| 服务注册与发现                           | <font color=red>Eureka</font>，Consul，ZooKeeper，<font color=green>Nacos</font> |      |
| 服务调用                                 | Ribbon，Rest，RPC，GRPC                                      |      |
| 服务熔断                                 | <font color=red>Hystrix</font>，Envoy，<font color=green>sentienl</font> |      |
| 负载均衡                                 | Ribbon，Nginx 等                                             |      |
| 服务接口调用（客户端调用服务的简化工具） | Feign等                                                      |      |
| 消息队列                                 | Zuul等                                                       |      |
| 服务配置中心管理                         | Zabbix，Negios，Metrics，Spectator，<font color=green>Nacos</font> |      |
| 服务路由（API 网管）                     | Zuul等                                                       |      |
| 服务监控                                 | Zabbix，Negios，Metrics，Spectator等                         |      |
| 全链路追踪                               | Zipkin,Brave,Dapper等                                        |      |
| 数据流操作开发包                         | Spring Cloud Stream（封装与Redis，Kafka等发送接受消息）      |      |
| 事件消息总线                             | Spring Cloud Bus，<font color=green>Nacos</font>             |      |
| 服务部署                                 | Docker，OpenStack，Kubernetes等                              |      |



主流 IT 公司：

阿里：Dubbo/HSF

京东JSF

新浪微博Motan

当当网 DubboX

| Netflix/Spring cloud | Netflix/Spring cloud                  | Motan                                                        | GRPC     | Thrift   | Dubbo/DubboX |
| -------------------- | ------------------------------------- | ------------------------------------------------------------ | -------- | -------- | ------------ |
| 功能定位             | 完整的微服务框架                      | RPC 框架，但组合了ZK或Consul，实现集群环境的基本的服务注册/发现 | RPC 框架 | RPC 框架 | 服务框架     |
| 支持Rest             | 是，Ribbon 支持多种可插拔的序列化选择 | 否                                                           | 否       | 否       | 否           |
| 支持RPC              | 否                                    | 是(Hession2)                                                 | 是       | 是       | 是           |
| 服务注册/发现        |                                       |                                                              |          |          |              |
| 负载均衡             | 是（服务端zuul+客户端Ribbon）         | 是（客户端）                                                 | 否       | 否       | 是（客户端） |
| 服务配置             |                                       |                                                              |          |          |              |
| 支持多语言           | 是(Rest 形式)                         | 否                                                           | 是       | 是       | 是（客户端） |
|                      |                                       |                                                              |          |          |              |
|                      |                                       |                                                              |          |          |              |



创建微服务模块步骤：

1. 建 module
2. 该 POM
3. 写 YML
4. 启动类
5. 业务类





业务类：

1. 写 SQL
2. entities
3. dao
4. service
5. controller

