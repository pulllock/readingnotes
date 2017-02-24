# 架构
Dubbo架构图：

![](dubbo-architecture.jpg)

Provider和Consumer是必须存在的，Provider是暴露服务的服务提供方，Consumer是调用远程服务的服务消费方，Registry是服务发现和注册中心，生产环境建议使用。Monitor是监控和统计用。

# 应用执行过程
应用大概的执行过程：

- 由容器启动服务提供者，根据协议绑定到配置的IP和端口上，如果已有服务绑定相同的IP和端口则跳过。
- 注册服务信息到注册中心
- 服务消费方启动，根据协议信息订阅注册中心里面的服务，注册中心将存活的服务地址通知到客户端，当有服务信息变更时，客户端可以获得变更信息。
- 消费方需要调用服务时，从内存中拿到上次通知的所有存活的服务地址，根据路由信息和负载均衡选择调用的服务地址，发起调用。
- 通过filter分别在客户端发送请求钱和服务端接收请求后，通过异步记录信息，发送到monitor做监控和统计。

# 注册中心
Dubbo支持四种注册中心，Multicast注册中心，Zookeeper，redis，Simple注册中心。

## Zookeeper
适用于生产环境。

集群中部署奇数个节点，Zookeeper挂掉一半的机器，集群就不可用。

过程：

- 服务提供者启动时，向providers目录下写入自己的url地址。
- 服务消费者启动时，订阅providers目录下的提供者url地址，向consumers目录下写入自己的url地址。
- 监控中心启动时，订阅目录下的所有提供者和消费者的url地址。

## redis
Redis过期数据，通过心跳方式检测脏数据，服务器时间必须相同，对服务器有一定压力。

数据结构：

- 使用Redis的key/map结构存储数据，主key为服务名和类型，map中的key为url地址，map中的value为过期时间，用于判断脏数据，脏数据由监控中心删除。

- 使用redis的publish/subscribe事件通知数据变更。通过事件的值来区分事件类型：register，unregister，subscribe，unsubscribe。普通消费者直接订阅指定服务者提供的key，只会收到指定服务的register和unregister时间。监控中心通过psubscribe功能订阅/dubbo/*，会收到所有服务的所有变更事件。

过程：

- 服务提供方启动时，向主key：`/dubbo/com.xxx.YyyService/providers`下添加当前提供者的地址，并向Channel：`/dubbo/com.xxx.YyyService/providers`发送register事件。
- 服务消费方启动时，从Channel：`/dubbo/com.xxx.YyyService/providers`订阅register和unregister事件，并向主key：`/dubbo/com.xxx.YyyService/consumers`下添加当前消费者地址。
- 服务消费方收到register和unregister事件后，从主key`/dubbo/com.xxx.YyyService/providers`下获取提供者地址列表。
- 服务监控中心启动时，从Channel：`/dubbo/*`订阅register和unregister，以及subscribe和unsubscribe事件。收到register和unregister事件后，从主key：`/dubbo/com.xxx.YyyService/providers`下获取提供者列表。收到subscribe和unsubscribe事件后，从主key：`/dubbo/com.xxx.YyyService/consumers`下获取消费者地址列表。

## multicast
不需要启动任何中心节点，只要广播地址一样，就可以相互发现。组播受网络结构限制，只适合小规模应用或开发阶段使用。

过程：

- 提供方启动时广播自己的地址。
- 消费方启动时广播订阅请求。
- 提供方收到订阅请求，单播自己的地址给订阅者，如果设置了unicast=false，则广播给订阅者。
- 消费方收到提供的地址，则连接该地址进行RPC调用。

## Simple注册中心
注册中心本身就是一个Dubbo服务，可以减少第三方依赖，使整体通讯方式一致。

# Dubbo的加载
Dubbo基于Spring的Schema扩展进行加载。

# 协议
## dubbo://
Dubbo缺省协议，采用单一长连接和NIO异步通讯，适合小数据量大并发的服务调用，以及消费者机器远大于服务提供者机器的情况。

不适合传送大数据量的服务，比如文件，视频等。

Dubbo默认为每服务每提供者每消费者使用单一长连接，如果数据量较大，可以使用多个链接：

```
<dubbo:protocol name="dubbo" connections="2" />
```

为防止被大量连接撑挂，可在服务提供方限制大接收连接数，以实现服务提供方的自我保护。

## rmi://
RMI协议采用JDK标准的java.rmi.*实现，采用阻塞式短连接和JDK标准序列化方式。

适用于传入传出参数数据包大小混合，消费者与提供者个数差不多，可传文件。

适用于常规远程服务方法调用，与原生RMI服务互操作。

## hessian://
用于集成Hessian的服务，Hessian底层采用http通讯，采用servlet暴露服务，Dubbo缺省内嵌Jetty作为服务器实现。

适用于传入传出参数数据包比较大，提供者比消费者个数多，提供者压力较大，可传文件。

适用于页面传输，文件传输，与原生hessian服务互相操作。

## http://
采用Spring的HttpInvoker实现，基于http表单的远程调用协议。

适用于传入传出参数数据包大小混合，提供者比消费者个数多，可用浏览器查看，可用表单或URL传入参数，暂不支持传文件。
适用于需同时给程序和浏览器js使用的服务。

## webservice://
基于CXF的实现。

适用于系统集成，跨语言调用。

## thrift：//
对thrift协议的支持，做了扩展。不能传递null值。

## memcached://
## redis://

