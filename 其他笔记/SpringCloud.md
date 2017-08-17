# 特性

- 分布式/版本化配置
- 服务的注册和发现
- 路由
- 服务到服务的调用
- 负载均衡
- 断路器
- 全局锁
- 领导选举和集群状态
- 分布式消息

# 主要项目
## Spring Cloud Config
由Git仓库支持的集中式外部配置管理。配置资源直接映射到Spring的`Environment`，如果需要还可以用于非Spring的应用。

## Spring Cloud Netflix
与各种Netflix OSS组件集成。Eureka, Hystrix, Zuul, Archaius等。



## Spring Cloud Bus
用于将服务和服务实例以及分布式消息连接的事件总线。用于通过集群传递状态改变，比如配置改变事件。

## Spring Cloud for Cloud Foundry
将应用与Pivotal Cloudfoundry集成。提供服务发现的实现，还可以轻松的实现SSO和OAuth2保护的资源，还可以创建Cloudfoundry服务代理。

## Spring Cloud Cloud Foundry Service Broker
提供构建服务代理的起点，用来管理Cloud Foundry管理的服务。

## Spring Cloud Cluster
对于Zookeeper，Redis，Hazelcast，Consul的抽象和实现。领导选举和共同的有状态

## Spring Cloud Consul
Hashicorp Consul管理的服务发现和配置管理。

## Spring Cloud Security
在Zuul代理中支持负载均衡的OAuth2 rest客户端和认证头转发。

## Spring Cloud Sleuth
Spring Cloud应用的分布式跟踪，和Zipkin，HTrace和基于log的跟踪（比如ELK）兼容。

## Spring Cloud Data Flow

用于现代运行时，可组合的微服务应用程序的云本地编排服务。易于使用的DSL，拖放式GUI和REST API一起简化了基于微服务的数据管道的整体编排。

## Spring Cloud Stream
轻量级的事件驱动的微服务框架，用于快速构建应用，并且可以连接到外部系统。使用Apache Kafka或RabbitMQ在Spring Boot应用程序之间发送和接收消息的简单声明模型。

## Spring Cloud Stream App Starters
Spring Cloud Stream应用程序启动器是基于Spring Boot的Spring集成应用程序，提供与外部系统的集成。

## Spring Cloud Task
一种短期的微服务框架，用于快速构建执行有限数据处理的应用程序。简单的声明，将功能和非功能功能添加到Spring Boot应用程序。

## Spring Cloud Task App Starters
Spring Cloud任务应用程序启动器是Spring引导应用程序，可能是任何进程，包括不会永久运行的Spring Batch作业，并且在有限的数据处理时间结束/停止。

## Spring Cloud Zookeeper
使用zookeeper进行服务的发现和配置管理。

## Spring Cloud for Amazon Web Services
与托管的Amazon Web Services轻松集成。它提供了一种方便的方式，使用众所周知的Spring惯用语和API（如消息传递或缓存API）与AWS提供的服务进行交互。开发人员可以围绕托管服务构建应用程序，而无需关心基础设施或维护。

## Spring Cloud Connectors
使PaaS应用程序在各种平台中轻松连接到后端服务，如数据库和消息代理（以前称为“Spring Cloud”）。

## Spring Cloud Starters
Spring Boot-style启动项目，以缓解Spring Cloud消费者的依赖管理。 （停止作为项目，并与Angel.SR2之后的其他项目合并。）

## Spring Cloud CLI
Spring Boot CLI插件用于在Groovy中快速创建Spring Cloud组件应用程序

## Spring Cloud Contract
Spring Cloud合同是一个总体项目，其中包含帮助用户成功实施消费者驱动合同方法的解决方案。

# Spring Cloud Netflix
Eureka 服务发现，注册，充当注册中心，使用@EnableEurekaServe注解

Hystrix 断路器

Zuul 路由

Ribbon Client端负载均衡

# 大概流程
- 首先需要一个放置配置文件的地方，可以在本地，也可以是git仓库。
- 然后是配置服务器，从放置配置文件的地方获取配置，提供给其他的系统用。
- 创建一个注册中心，使用Eureka，提供服务的注册和发现，@EnableEurekaServer
- 创建服务提供者，@EnableDiscoveryClient
- 创建服务消费者，@EnableDiscoveryClient

## Eureka服务注册和发现模块
`@EnableEurekaServer`启用一个服务注册中心

Eureka server默认情况下也是一个client，所以必须要指定一个是server，一般配置如下：

```
server:
  port: 8761

eureka:
  client:
    registerWithEureka: false
    fetchRegistry: false
  server:
    waitTimeInMsWhenSyncEmpty: 0
```
当`eureka.client.registerWithEurka = false`和`fetchRegistry = false`时，说明是eureka server。

`@EnableEurekaClient`启用一个eureka client，服务提供者

## ribbon服务消费者
ribbon是负载均衡客户端

`@EnableDiscoveryClient`可以向服务中心注册

`@LoadBalanced`可以开启负载均衡功能

## Fegin服务消费者
`@EnableFeignClients`开启Fegin功能

Fegin是一个声明式的伪Http客户端，默认集成了Ribbon，默认实现了负载均衡的效果。

`@FeginClient("service name")`来指定调用哪个服务

## Hystrix断路由
`@EnableHystrix`可以启动断路由

`@HystrixCommand(fallbackMethod="xxx")`注解在服务上

Fegin中默认启用了断路由

`@EnableHystrixDashboard`可以启用断路由的仪表盘

## Zuul路由网关
Zuul主要功能是路由和过滤器，实现了负载均衡

`@EnavleZuulProxy`开启

## Spring Cloud Config分布式配置中心
Spring Cloud Config提供一个服务端和客户端，来提供可扩展的配置服务

`@EnableConfigServer`开启配置服务器

## Spring Cloud Bus消息总线
Spring Cloud Bus可以讲分布式节点和轻量的消息代理连接起来
