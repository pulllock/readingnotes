从头开始写一个迷你的dubbo，仅用作学习用，学习的过程中更深入的了解下dubbo，同时也补充下其他的知识。

## 工程说明
## mini-dubbo-provider
服务提供者端，主要是暴露服务，处理消费者端请求等

## mini-dubbo-consumer
服务消费者端，主要作用是引用服务

## mini-dubbo-common
一些公用类

## mini-dubbo-sample-*
sample是示例项目

## 目标

- 可以使用API和Spring两种方式启动
- 提供完整暴露服务和服务引用功能
- 提供多协议支持
- 实现注册中心支持
- 提供自定义编解码
- 使用Netty和Mina
- 提供对多种序列化的支持

## 已有实现
- 可以使用API启动
- 简单的暴露和服务引用
- 现使用TCP协议
- 使用Netty
- 使用Netty的编解码
- 使用Java序列化方式

## 源码地址
[https://github.com/dachengxi/mini-dubbo](https://github.com/dachengxi/mini-dubbo)

# 过程
现在的版本就是一个简单的RPC调用功能。下面是大概的过程：

1. 服务提供者端DubboProvider，这是API，用来启动服务暴露的功能，会调用Netty实现的服务端暴露服务，并监听处理。
2. 编写Netty服务和Handler。
3. 实现请求和相应分别对应的两个bean：Request和Response。
4. 服务消费者端DubboConsumer，这是API，用来初始化消费者，和调用服务提供者。实际应该是先获取代理，在真正使用的时候采取调用服务提供者端，现在都在一步中完成了。
5. 编写获取代理类DubboConsumerProxy。
6. 编写Netty客户端和Handler。
7. 测试都在sample的模块下。

还有很多要做的～加油！！