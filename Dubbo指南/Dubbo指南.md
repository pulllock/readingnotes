# Dubbo用户指南

# 架构
![](dubbo-architecture.jpg)

## 节点角色

- Provider 暴露服务，服务提供方。
- Consumer 调用远程服务，服务消费方。
- Registry 服务注册与发现的注册中心。
- Monitor 统计服务调用次数，调用时间的监控中心。
- Container 服务运行容器。

# 用法
 
 Spring配置：
 
 提供方暴露服务配置`<dubbo:service>`
 
 消费方引用服务配置`<dubbo:reference>`
 
 Dubbo基于Spring的Schema扩展进行加载。
 
# 启动

## 服务提供者
provider.xml

```

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans        		http://www.springframework.org/schema/beans/spring-beans.xsd        		http://code.alibabatech.com/schema/dubbo
    	http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
 
    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="hello-world-app"  />
    <!-- 使用multicast广播注册中心暴露服务地址 -->
    <dubbo:registry address="multicast://224.5.6.7:1234" />
    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880" />
    <!-- 声明需要暴露的服务接口 -->
    <dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService" />
    <!-- 和本地bean一样实现服务 -->
    <bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl" />
</beans>
```

## 服务消费者

consumer.xml

```

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans        		http://www.springframework.org/schema/beans/spring-beans.xsd        		http://code.alibabatech.com/schema/dubbo
    	http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
    <dubbo:application name="consumer-of-helloworld-app"  />
    <!-- 使用multicast广播注册中心暴露发现服务地址 -->
    <dubbo:registry address="multicast://224.5.6.7:1234" />
    <!-- 生成远程服务代理，可以和本地bean一样使用demoService -->
    <dubbo:reference id="demoService" interface="com.alibaba.dubbo.demo.DemoService" />
 
</beans>
```

# 配置

## xml配置

- `<dubbo:service/>` 服务配置，用于暴露一个服务，定义服务的元信息，一个服务可以用多个协议暴露，一个服务也可以注册到多个注册中心。
- <dubbo:reference/> 引用配置，用于创建一个远程服务代理，一个引用可以指向多个注册中心。
- <dubbo:protocol/> 协议配置，用于配置提供服务的协议信息，协议由提供方指定，消费方被动接受。
- <dubbo:application/> 应用配置，用于配置当前应用信息，不管该应用是提供者还是消费者。
- <dubbo:module/> 模块配置，用于配置当前模块信息，可选。
- <dubbo:registry/> 注册中心配置，用于配置连接注册中心相关信息。
- <dubbo:monitor/> 监控中心配置，用于配置连接监控中心相关信息，可选。
- <dubbo:provider/> 提供方的缺省值，当ProtocolConfig和ServiceConfig某属性没有配置时，采用此缺省值，可选。
- <dubbo:consumer/> 消费方缺省配置，当ReferenceConfig某属性没有配置时，采用此缺省值，可选。
- <dubbo:method/> 方法配置，用于ServiceConfig和ReferenceConfig指定方法级的配置信息。
- <dubbo:argument/> 用于指定方法参数配置。

配置的查找顺序：

- 方法级优先，接口级次之，全局配置再次之。
- 如果级别一样，则消费方优先，提供方次之。

## 属性配置

- 如果公共配置很简单，没有多注册中心，多协议等情况，或者想多个Spring容器想共享配置，可以使用dubbo.properties作为缺省配置。
- Dubbo将自动加载classpath根目录下的dubbo.properties，可以通过JVM启动参数：`-Ddubbo.properties.file=xxx.properties`改变缺省配置位置。

### 映射规则：

- 将XML配置的标签名，加属性名，用点分隔，多个属性拆成多行：
比如：`dubbo.application.name=foo`等价于`<dubbo:application name="foo" />`
比如：`dubbo.registry.address=10.20.153.10:9090`等价于`<dubbo:registry address="10.20.153.10:9090" />`
- 如果XML有多行同名标签配置，可用id号区分，如果没有id号将对所有同名标签生效：
比如：`dubbo.protocol.rmi.port=1234`等价于`<dubbo:protocol id="rmi" name="rmi" port="1099" />` (协议的id没配时，缺省使用协议名作为id)
比如：`dubbo.registry.china.address=10.20.153.10:9090`等价于`<dubbo:registry id="china" address="10.20.153.10:9090" />`

### 覆盖策略：

- JVM启动-D参数优先，这样可以使用户在部署和启动时进行参数重写，比如在启动时需改变协议的端口。
- XML次之，如果在XML中有配置，则dubbo.properties中的相应配置项无效。
- Properties最后，相当于缺省值，只有XML没有配置时，dubbo.properties的相应配置项才会生效，通常用于共享公共配置，比如应用名。

## 注解配置

### 服务提供方注解：

```
@Service(version="1.0.0")
```

### 服务提供方配置：

```
<!-- 公共信息，也可以用dubbo.properties配置 -->
<dubbo:application name="annotation-provider" />
<dubbo:registry address="127.0.0.1:4548" />
 
<!-- 扫描注解包路径，多个包用逗号分隔，不填pacakge表示扫描当前ApplicationContext中所有的类 -->
<dubbo:annotation package="com.foo.bar.service" />
```

### 服务消费方注解：

```
 @Reference(version="1.0.0")
  private FooService fooService;
```

### 服务消费方配置：

```
<!-- 公共信息，也可以用dubbo.properties配置 -->
<dubbo:application name="annotation-consumer" />
<dubbo:registry address="127.0.0.1:4548" />
 
<!-- 扫描注解包路径，多个包用逗号分隔，不填pacakge表示扫描当前ApplicationContext中所有的类 -->
<dubbo:annotation package="com.foo.bar.action" />
```

## API配置
API仅用于OpenAPI, ESB, Test, Mock等系统集成。

### 服务提供方：

```
// 服务实现
XxxService xxxService = new XxxServiceImpl();
 
// 当前应用配置
ApplicationConfig application = new ApplicationConfig();
application.setName("xxx");
 
// 连接注册中心配置
RegistryConfig registry = new RegistryConfig();
registry.setAddress("10.20.130.230:9090");
registry.setUsername("aaa");
registry.setPassword("bbb");
 
// 服务提供者协议配置
ProtocolConfig protocol = new ProtocolConfig();
protocol.setName("dubbo");
protocol.setPort(12345);
protocol.setThreads(200);
 
// 注意：ServiceConfig为重对象，内部封装了与注册中心的连接，以及开启服务端口
 
// 服务提供者暴露服务配置
ServiceConfig<XxxService> service = new ServiceConfig<XxxService>(); // 此实例很重，封装了与注册中心的连接，请自行缓存，否则可能造成内存和连接泄漏
service.setApplication(application);
service.setRegistry(registry); // 多个注册中心可以用setRegistries()
service.setProtocol(protocol); // 多个协议可以用setProtocols()
service.setInterface(XxxService.class);
service.setRef(xxxService);
service.setVersion("1.0.0");
 
// 暴露及注册服务
service.export();
```

### 服务消费者：

```
// 当前应用配置
ApplicationConfig application = new ApplicationConfig();
application.setName("yyy");
 
// 连接注册中心配置
RegistryConfig registry = new RegistryConfig();
registry.setAddress("10.20.130.230:9090");
registry.setUsername("aaa");
registry.setPassword("bbb");
 
// 注意：ReferenceConfig为重对象，内部封装了与注册中心的连接，以及与服务提供方的连接
 
// 引用远程服务
ReferenceConfig<XxxService> reference = new ReferenceConfig<XxxService>(); // 此实例很重，封装了与注册中心的连接以及与提供者的连接，请自行缓存，否则可能造成内存和连接泄漏
reference.setApplication(application);
reference.setRegistry(registry); // 多个注册中心可以用setRegistries()
reference.setInterface(XxxService.class);
reference.setVersion("1.0.0");
 
// 和本地bean一样使用xxxService
XxxService xxxService = reference.get(); // 注意：此代理对象内部封装了所有通讯细节，对象较重，请缓存复用
```

### 特殊场景

#### 方法级别设置

```
...
 
// 方法级配置
List<MethodConfig> methods = new ArrayList<MethodConfig>();
MethodConfig method = new MethodConfig();
method.setName("createXxx");
method.setTimeout(10000);
method.setRetries(0);
methods.add(method);
 
// 引用远程服务
ReferenceConfig<XxxService> reference = new ReferenceConfig<XxxService>(); // 此实例很重，封装了与注册中心的连接以及与提供者的连接，请自行缓存，否则可能造成内存和连接泄漏
...
reference.setMethods(methods); // 设置方法级配置
 
...
```

#### 点对点直连

```
...
 
ReferenceConfig<XxxService> reference = new ReferenceConfig<XxxService>(); // 此实例很重，封装了与注册中心的连接以及与提供者的连接，请自行缓存，否则可能造成内存和连接泄漏
// 如果点对点直连，可以用reference.setUrl()指定目标地址，设置url后将绕过注册中心，
// 其中，协议对应provider.setProtocol()的值，端口对应provider.setPort()的值，
// 路径对应service.setPath()的值，如果未设置path，缺省path为接口名
reference.setUrl("dubbo://10.20.130.230:20880/com.xxx.XxxService"); 
 
...

```

# 启动时检查
Dubbo缺省会在启动时检查依赖的服务是否可用，不可用会抛异常，阻止Spring初始化完成，以便能及早发现问题，默认check=true。

如果Spring容器是懒加载的，或者通过API编程延迟引用服务，关闭check，否则服务临时不可用时，会抛出异常，拿到null引用，如果check=false，总是会返回引用，当服务恢复时，能自动连上。

- 关闭某个服务的启动时检查：(没有提供者时报错)
	
	```
	<dubbo:reference interface="com.foo.BarService" check="false" />
	```
- 关闭所有服务的启动时检查：(没有提供者时报错)
	
	```
	<dubbo:consumer check="false" />
	```
- 关闭注册中心启动时检查：(注册订阅失败时报错)
	
	```
	<dubbo:registry check="false" />
	```
也可以用dubbo.properties配置：

```
dubbo.reference.com.foo.BarService.check=false
dubbo.reference.check=false
dubbo.consumer.check=false
dubbo.registry.check=false
```

也可以用-D参数：

```
java -Ddubbo.reference.com.foo.BarService.check=false
java -Ddubbo.reference.check=false
java -Ddubbo.consumer.check=false 
java -Ddubbo.registry.check=false
```

引用缺省是延迟初始化的，只有引用被注入到其它Bean，或被getBean()获取，才会初始化。
如果需要饥饿加载，即没有人引用也立即生成动态代理，可以配置：

```
<dubbo:reference interface="com.foo.BarService" init="true" />
```

# 集群容错
集群调用失败时，Dubbo提供了多种容错方案，缺省位failover重试。

- Failover Cluster
	
	+ 失败自动切换，当出现失败，重试其它服务器。(缺省)
	+ 通常用于读操作，但重试会带来更长延迟。
	+ 可通过retries="2"来设置重试次数(不含第一次)。

- Failfast Cluster

	+ 快速失败，只发起一次调用，失败立即报错。
	+ 通常用于非幂等性的写操作，比如新增记录。

- Failsafe Cluster
	
	+ 失败安全，出现异常时，直接忽略。
	+ 通常用于写入审计日志等操作。

- Failback Cluster
	
	+ 失败自动恢复，后台记录失败请求，定时重发。
	+ 通常用于消息通知操作。

- Forking Cluster
	
	+ 并行调用多个服务器，只要一个成功即返回。
	+ 通常用于实时性要求较高的读操作，但需要浪费更多服务资源。
	+ 可通过forks="2"来设置最大并行数。

- Broadcast Cluster
	
	+ 广播调用所有提供者，逐个调用，任意一台报错则报错。(2.1.0开始支持)
	+ 通常用于通知所有提供者更新缓存或日志等本地资源信息。

重试次数配置如：(failover集群模式生效)

```
<dubbo:service retries="2" />

<dubbo:reference retries="2" />

<dubbo:reference>
    <dubbo:method name="findFoo" retries="2" />
</dubbo:reference>
```

集群模式配置如：

```
<dubbo:service cluster="failsafe" />


<dubbo:reference cluster="failsafe" />
```

# 负载均衡
缺省为random随机调用。

+ Random LoadBalance
	
	- 随机，按权重设置随机概率。
	- 在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重。

+ RoundRobin LoadBalance

	- 轮循，按公约后的权重设置轮循比率。
	- 存在慢的提供者累积请求问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上。

+ LeastActive LoadBalance
	
	- 最少活跃调用数，相同活跃数的随机，活跃数指调用前后计数差。
	- 使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大。

+ ConsistentHash LoadBalance
	
	- 一致性Hash，相同参数的请求总是发到同一提供者。
	- 当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。
	- 缺省只对第一个参数Hash，如果要修改，请配置`<dubbo:parameter key="hash.arguments" value="0,1" />`
	- 缺省用160份虚拟节点，如果要修改，请配置`<dubbo:parameter key="hash.nodes" value="320" />`

配置如：

```
<dubbo:service interface="..." loadbalance="roundrobin" />

<dubbo:reference interface="..." loadbalance="roundrobin" />

<dubbo:service interface="...">
    <dubbo:method name="..." loadbalance="roundrobin"/>
</dubbo:service>

<dubbo:reference interface="...">
    <dubbo:method name="..." loadbalance="roundrobin"/>
</dubbo:reference>
```

# 线程模型

+ Dispatcher

	- all 所有消息都派发到线程池，包括请求，响应，连接事件，断开事件，心跳等。
	- direct 所有消息都不派发到线程池，全部在IO线程上直接执行。
	- message 只有请求响应消息派发到线程池，其它连接断开事件，心跳等消息，直接在IO线程上执行。
	- execution 只请求消息派发到线程池，不含响应，响应和其它连接断开事件，心跳等消息，直接在IO线程上执行。
	- connection 在IO线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池。
	
+ ThreadPool
	- fixed 固定大小线程池，启动时建立线程，不关闭，一直持有。(缺省)
	- cached 缓存线程池，空闲一分钟自动删除，需要时重建。
	- limited 可伸缩线程池，但池中的线程数只会增长不会收缩。(为避免收缩时突然来了大流量引起的性能问题)。

配置如：

```
<dubbo:protocol name="dubbo" dispatcher="all" threadpool="fixed" threads="100" />
```

# 只订阅
禁止注册配置：

```
<dubbo:registry address="10.20.153.10:9090" register="false" />

<dubbo:registry address="10.20.153.10:9090?register=false" />
```

# 只注册
禁止订阅配置：

```
<dubbo:registry id="qdRegistry" address="10.20.141.150:9090" subscribe="false" />

<dubbo:registry id="qdRegistry" address="10.20.141.150:9090?subscribe=false" />
```

# 静态服务

```
<dubbo:registry address="10.20.141.150:9090" dynamic="false" />

<dubbo:registry address="10.20.141.150:9090?dynamic=false" />
```

# 多协议

## 不同服务 不同协议

```
    <!-- 多协议配置 -->
    <dubbo:protocol name="dubbo" port="20880" />
    <dubbo:protocol name="rmi" port="1099" />
 
    <!-- 使用dubbo协议暴露服务 -->
    <dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService" protocol="dubbo" />
    <!-- 使用rmi协议暴露服务 -->
    <dubbo:service interface="com.alibaba.hello.api.DemoService" version="1.0.0" ref="demoService" protocol="rmi" />
```

## 多协议暴露服务

```
 <!-- 多协议配置 -->
    <dubbo:protocol name="dubbo" port="20880" />
    <dubbo:protocol name="hessian" port="8080" />
 
    <!-- 使用多个协议暴露服务 -->
    <dubbo:service id="helloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" protocol="dubbo,hessian" />
```

# 多注册中心

## 多注册中心注册

```
<!-- 多注册中心配置 -->
    <dubbo:registry id="hangzhouRegistry" address="10.20.141.150:9090" />
    <dubbo:registry id="qingdaoRegistry" address="10.20.141.151:9010" default="false" />
 
    <!-- 向多个注册中心注册 -->
    <dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService" registry="hangzhouRegistry,qingdaoRegistry" />

```

## 不同服务使用不同注册中心

```
<!-- 多注册中心配置 -->
    <dubbo:registry id="chinaRegistry" address="10.20.141.150:9090" />
    <dubbo:registry id="intlRegistry" address="10.20.154.177:9010" default="false" />
 
    <!-- 向中文站注册中心注册 -->
    <dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService" registry="chinaRegistry" />
 
    <!-- 向国际站注册中心注册 -->
    <dubbo:service interface="com.alibaba.hello.api.DemoService" version="1.0.0" ref="demoService" registry="intlRegistry" />
 
```

## 多注册中心引用

```
 <!-- 多注册中心配置 -->
    <dubbo:registry id="chinaRegistry" address="10.20.141.150:9090" />
    <dubbo:registry id="intlRegistry" address="10.20.154.177:9010" default="false" />
 
    <!-- 引用中文站服务 -->
    <dubbo:reference id="chinaHelloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" registry="chinaRegistry" />
 
    <!-- 引用国际站站服务 -->
    <dubbo:reference id="intlHelloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" registry="intlRegistry" />

或者：

<!-- 多注册中心配置，竖号分隔表示同时连接多个不同注册中心，同一注册中心的多个集群地址用逗号分隔 -->
    <dubbo:registry address="10.20.141.150:9090|10.20.154.177:9010" />
 
    <!-- 引用服务 -->
    <dubbo:reference id="helloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" />
```

# 服务分组
# 多版本
# 分组聚合
# 参数验证

# 结果缓存
结果缓存，用于加速热门数据的访问速度，Dubbo提供声明式缓存，以减少用户加缓存的工作量。

- lru 基于最近最少使用原则删除多余缓存，保持最热的数据被缓存。
- threadlocal 当前线程缓存，比如一个页面渲染，用到很多portal，每个portal都要去查用户信息，通过线程缓存，可以减少这种多余访问。
- jcache 与JSR107集成，可以桥接各种缓存实现。

```
配置如：

<dubbo:reference interface="com.foo.BarService" cache="lru" />
或：

<dubbo:reference interface="com.foo.BarService">
    <dubbo:method name="findBar" cache="lru" />
</dubbo:reference>
```

# 异步调用
基于NIO的非阻塞实现并行调用，客户端不需要启动多线程即可完成并行调用多个远程服务，相对多线程开销较小。

async="true"

# 本地调用
本地调用，使用了Injvm协议，是一个伪协议，它不开启端口，不发起远程调用，只在JVM内直接关联，但执行Dubbo的Filter链。

# Reference Config缓存
ReferenceConfig实例很重，封装了与注册中心的连接以及与提供者的连接，需要缓存，否则重复生成ReferenceConfig可能造成性能问题并且会有内存和连接泄漏。API方式编程时，容易忽略此问题。

提供了简单的工具类ReferenceConfigCache用于缓存ReferenceConfig实例。

```

ReferenceConfig<XxxService> reference = new ReferenceConfig<XxxService>();
reference.setInterface(XxxService.class);
reference.setVersion("1.0.0");
......
 
 
ReferenceConfigCache cache = ReferenceConfigCache.getCache();
XxxService xxxService = cache.get(reference); // cache.get方法中会Cache Reference对象，并且调用ReferenceConfig.get方法启动ReferenceConfig
// 注意！ Cache会持有ReferenceConfig，不要在外部再调用ReferenceConfig的destroy方法，导致Cache内的ReferenceConfig失效！
 
// 使用xxxService对象
xxxService.sayHello();
```

消除Cache中的ReferenceConfig，销毁ReferenceConfig并释放对应的资源。



```
eferenceConfigCache cache = ReferenceConfigCache.getCache();
cache.destroy(reference);
```

# 协议参考

## dubbo://
Dubbo缺省协议采用单一长连接和NIO异步通讯，适合于小数据量大并发的服务调用，以及服务消费者机器数远大于服务提供者机器数的情况。

Dubbo缺省协议不适合传送大数据量的服务，比如传文件，传视频等，除非请求量很低。

配置：

dubbo.properties

```
dubbo.service.protocol=dubbo
```

## rmi://
RMI协议采用JDK标准的java.rmi.*实现，采用阻塞式短连接和JDK标准序列化方式。

## hessian://
Hessian协议用于集成Hessian的服务，Hessian底层采用Http通讯，采用Servlet暴露服务，Dubbo缺省内嵌Jetty作为服务器实现。

## http://
采用Spring的HttpInvoker实现

## webservice://
基于CXF的frontend-simple和transports-http实现。

## thrift://
Thrift是Facebook捐给Apache的一个RPC框架

## memcached://
可以通过脚本或监控中心手工填写表单注册memcached服务的地址。

## redis://
可以通过脚本或监控中心手工填写表单注册redis服务的地址。

# 注册中心
推荐使用Zookeeper注册中心

## Multicast注册中心
不需要启动任何中心节点，只要广播地址一样，就可以互相发现

## Zookeeper注册中心

## Redis注册中心

## Simple注册中心
注册中心本身就是一个普通的Dubbo服务，可以减少第三方依赖，使整体通讯方式一致。

此SimpleRegistryService只是简单实现，不支持集群，可作为自定义注册中心的参考，但不适合直接用于生产环境。
