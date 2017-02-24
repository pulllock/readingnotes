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

# dubbo源码学习
## dubbo架构设计
架构图：

![](dubbo-architecture.png)

另外一张：

![](dubbo-framework.jpg)

左边淡蓝色的为消费方使用的接口，右边淡绿色的是服务提供方使用的接口，中轴线为双方都使用到的接口。

从上到下共分10层，均为单向依赖，每一层的上层都能被剥离复用。Service和Config层为API，其它层为SPI。

蓝色虚线为初始化过程，即启动时组装链，红色实线为方法调用过程，即运行时调用链，紫色为继承。

### API与SPI
API，Application Programming Interface，直接被应用开发人员使用，

SPI，Service Provider Interface，被框架开发人员使用。

## 各层说明

1. Service，服务接口，与实际业务逻辑相关，根据服务提供方和服务消费方的业务设计对应的接口和实现。
2. Config，配置层，对外配置接口，以ServiceConfig和ReferenceConfig为中心，可以直接new配置类，也可以通过Spring解析配置生成配置类。
3. Proxy，服务代理层，服务接口透明代理，生产服务的客户端Stub和服务端Skeleton，以ServiceProxy为中心，扩展接口为ProxyFactory。
4. Registry，注册中心层，封装服务地址的注册和发现，以服务URL为中心，扩展接口为RegistryFactory，Registry和RegistryService。可能没有服务注册中心，此时服务提供方直接暴露服务。
5. Cluster，路由层，封装多个提供者的路由以及负载均衡，并桥接注册中心，以Invoker为中心，扩展接口为Cluster，Directory，Router，LoadBalance。将多个服务提供方组合为一个服务提供方，实现对服务消费者透明，只需要与一个服务提供方交互。
6. Monitor，监控层，RPC调用次数和调用时间监控，以Staticstics为中心，扩展接口为MonitorFactory，Monitor和MonitorService。
7. Protocol，远程调用层，封装RPC调用，以Invocation和Result为中心，扩展接口为Protocol，Invoke和Exporter。Protocol是服务域，它是Invoker暴露和引用的主功能入口，它负责Invoker的生命周期管理。Invoker是实体域，是dubbo的核心模型其他模型都向他靠拢或转换成他，他代表一个可执行体，可以向他发起一个invoke调用，有可能是一个本地实现，也可能是一个远程的实现，也可能是一个集群实现。
8. Exchange，信息交换层，封装请求相应模式，同步转异步，以Request和Response为中心，扩展接口为Exchanger，ExchangeChannel，ExchangeClient和ExchangeServer。
9. Transport，网络传输层，抽象mina和netty为统一接口，以Message为中心，扩展接口为Channel，Transporter，Client，Server和Codec。
10. Serialize，数据序列化层，可复用的一些工具，扩展接口为Serialization，ObjectInput，ObjectOutput和ThreadPool。

## Java中RPC介绍
RPC，Remote Procedure Call，远程过程调用。调用过程代码不是在调用者本地运行。

RPC调用流程：

1. 服务消费方调用，以本地调用方式调用服务。
2. Client Stub接收到调用请求之后，负责将方法，参数等组装成能够进行网络传输的消息体。
3. Client Stub找到服务地址，并将消息发送到服务端。
4. Server Stub收到消息后进行解码。
5. Server Stub根据解码结果调用本地的服务。
6. 本地服务将执行结果返回给Server Stub。
7. Server Stub将返回结果打包成消息并发送至消费方。
8. Client Stub接收到消息，并进行解码。
9. 服务消费方得到最终结果。

RPC目标就是要把2-8步骤封装起来，对用户透明。

### RMI
是RPC的Java版本，采用JRMP通讯协议，构建在TCP/IP协议上的一种远程调用方法。

RMI采用stubs和skeletons来进行远程对象的通讯，stub是远程对象的客户端代理有着和远程对象相同的远程接口。远程对象的调用实际是通过调用该对象的客户端代理对象stub来完成的。

创建远程方法调用的步骤：

1. 创建远程接口以及声明远程方法，这是实现双方通信的接口，需要继承Remote。
2. 开发远程接口的实现类以及远程方法，实现类需要集成UnicastRemoteObject。
3. 启动远程对象。
4. 客户端查找远程对象，调用远程方法。

代码如下：

User对象：

```
import java.io.Serializable;

/**
 * Created by cheng.xi on 17-2-24.
 * 对象需要实现Serializable接口
 */
public class User implements Serializable{
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

UserService以及UserServiceImpl：

```
import java.rmi.Remote;
import java.rmi.RemoteException;

/**
 * Created by cheng.xi on 17-2-24.
 */
public interface UserService extends Remote {
    User getUserByName(String name) throws RemoteException;
}
```

```
import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;

/**
 * Created by cheng.xi on 17-2-24.
 */
public class UserServiceImpl extends UnicastRemoteObject implements UserService{
    protected UserServiceImpl() throws RemoteException {
        super();
    }

    @Override
    public User getUserByName(String name) throws RemoteException {
        System.out.println("服务端方法开始被调用了");
        User user = new User();
        user.setName("H-" + name);
        return user;
    }
}
```

Server.class，发布服务，直接启动main方法即可：

```
import java.net.MalformedURLException;
import java.rmi.Naming;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;

/**
 * Created by cheng.xi on 17-2-24.
 */
public class Server {
    public static void main(String[] args) {
        try {
            UserService userService = new UserServiceImpl();
            //注册通讯端口
            LocateRegistry.createRegistry(8888);
            //注册service
            Naming.rebind("rmi://127.0.0.1:8888/UserService",userService);
            System.out.println("服务端已启动，可以远程调用UserService");
        } catch (RemoteException e) {
            e.printStackTrace();
        } catch (MalformedURLException e) {
            e.printStackTrace();
        }
    }
}
```

Client.class，启动完Server中的main方法后，再启动Client：

```
package cx.test.rpc.test2;

import java.net.MalformedURLException;
import java.rmi.Naming;
import java.rmi.NotBoundException;
import java.rmi.RemoteException;

/**
 * Created by cheng.xi on 17-2-24.
 */
public class Client {
    public static void main(String[] args) {
        //调用远程对象
        try {
            UserService userService = (UserService) Naming.lookup("rmi://127.0.0.1:8888/UserService");
            System.out.println(userService.getUserByName("xiaoming").getName());
        } catch (NotBoundException e) {
            e.printStackTrace();
        } catch (MalformedURLException e) {
            e.printStackTrace();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
}

```
运行即可显示，还可以部署到其他的机器上，在局域网中尝试运行。