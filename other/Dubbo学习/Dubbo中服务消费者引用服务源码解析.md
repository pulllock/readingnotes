这里开始分析下dubbo中服务消费者初始化的源码，还是以一个示例开始，逐步去分析消费者初始化的过程。如果有看过dubbo中服务提供者初始化的源码分析那篇，可能会对这里的分析有点帮助，有些重复的东西不再做过多说明。如果服务提供者初始化能看明白，消费者初始化的过程也不会很难。

# 示例代码

这里只贴出一点重要的代码，完整的代码请参考Github仓库：[DubboTest](https://github.com/dachengxi/DubboTest)

dubbo-consumer.xml:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo
       http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!--当前应用信息，name是当前应用的名称，用于注册中心计算依赖关系，消费方和提供方的名称不要一样-->
    <dubbo:application name="dubbo-consumer" />

    <!--注册中心配置，protocol注册中心的地址协议，address注册中心服务器地址-->
    <dubbo:registry  protocol="zookeeper" address="127.0.0.1:2181" />

    <!--服务消费者引用服务配置，id服务引用Bean id，interface服务接口名-->
    <dubbo:reference id="helloService" interface="me.cxis.dubbo.service.HelloService" timeout="100000"/>

</beans>
```

启动Consumer代码：

```java
public class StartConsumer {

    public static void main(String[] args) {
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext(new String[]{"dubbo-consumer.xml"});
        applicationContext.start();

        System.out.println("消费者端已经启动。");

        HelloService helloService = (HelloService) applicationContext.getBean("helloService");
        System.out.println("获取完bean");
        helloService.sayHello();

        System.out.println("按任意键退出");
        try {
            System.in.read();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

# 开始初始化的时机

dubbo服务消费者初始化的时机有两种：

- 配置了init=true。
- 没有配置或者init=false

如果说没有配置或者init=false，表示服务是延迟进行初始化的，这种情况下不会在实例化Bean的时候直接进行dubbo消费者的初始化，而是在实际使用到这个服务的时候进行初始化。我们可以看ReferenceBean实现了FactoryBean接口，当使用Spring获取一个Bean的时候，实际上会`ReferenceBean<T>`的`getObject()`方法，也就是在这个方法中dubbo进行了消费者的初始化操作。

如果配置了init=true，表示在Bean初始化的时候就要执行dubbo消费者初始化流程。ReferenceBean实现了InitializingBean接口，当Spring的Bean实例化的时候，会回调实现了InitializingBean的afterPropertiesSet方法，也就是会调用`ReferenceBean<T>`的afterPropertiesSet方法，在此方法中进行消费者初始化的操作。

默认情况都是使用延迟初始化，如果有特殊需求可以配置init属性，进行饿汉式的立即初始化。我们下面的流程就按照默认的配置来进行分析。前置的一些代码不在进行分析，直接从init方法开始看。

# 初始化前的各种配置

ReferenceConfig的init方法：

```java
private void init() {
    ...
    ...
    ref = createProxy(map);
}
```

代码比较多，主要的作用和提供者初始化的时候一样，也是进行各种属性检查，处理填充默认属性，处理generic泛化调用的接口等等处理。最后调用createProxy进行代理的创建。

# 开始初始化服务引用

服务的引用有三种形式：

- 本地JVM服务。
- 直连进行远程服务的引用。
- 通过注册中心进行远程服务的引用。

我们只看通过注册中心进行远程服务的引用，其他两种也都类似。我们看下createProxy的代码：

```java
private T createProxy(Map<String, String> map) {
    URL tmpUrl = new URL("temp", "localhost", 0, map);
    final boolean isJvmRefer;
    if (isInjvm() == null) {
        // 指定URL的情况下，不做本地引用
        if (url != null && url.length() > 0) {
            isJvmRefer = false;
        } else if (InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl)) {
            // 默认情况下如果本地有服务暴露，则引用本地服务.
            isJvmRefer = true;
        } else {
            isJvmRefer = false;
        }
    } else {
        isJvmRefer = isInjvm().booleanValue();
    }

    // 本地引用
    if (isJvmRefer) {
        // 本地应用的URL，就是injvm://开头的
        URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
        // 创建InjvmInvoker实例
        invoker = refprotocol.refer(interfaceClass, url);
        if (logger.isInfoEnabled()) {
            logger.info("Using injvm service " + interfaceClass.getName());
        }
    } else {
        // 远程引用
        // 用户指定URL，指定的URL可能是对点对直连地址，也可能是注册中心URL
        if (url != null && url.length() > 0) {
            String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
            // 可能会配置多个URL，分号分隔的
            if (us != null && us.length > 0) {
                for (String u : us) {
                    URL url = URL.valueOf(u);
                    if (url.getPath() == null || url.getPath().length() == 0) {
                        url = url.setPath(interfaceName);
                    }

                    // registry协议的，说明是要使用指定的注册中心
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                    } else {
                        // 合并url
                        urls.add(ClusterUtils.mergeUrl(url, map));
                    }
                }
            }
        } else {
            // 加载注册中心的URL
            List<URL> us = loadRegistries(false);
            if (us != null && us.size() > 0) {
                for (URL u : us) {
                    URL monitorUrl = loadMonitor(u);
                    if (monitorUrl != null) {
                        map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                    }
                    urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                }
            }
            if (urls == null || urls.size() == 0) {
                throw new IllegalStateException("No such any registry to reference " + interfaceName  + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
            }
        }

        // 可能是单个注册中心，也可能是单个服务提供者
        if (urls.size() == 1) {
            // 构建Invoker实例
            invoker = refprotocol.refer(interfaceClass, urls.get(0));
        } else {
            // 多个注册中心或者多个服务提供者
            List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
            URL registryURL = null;
            // 得到所有的Invoker
            for (URL url : urls) {
                invokers.add(refprotocol.refer(interfaceClass, url));
                if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                    // 用了最后一个registry url
                    registryURL = url;
                }
            }
            // 有注册中心协议的URL
            if (registryURL != null) {
                // 对有注册中心的Cluster使用 AvailableCluster
                URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
                /**
                     * 创建StaticDirectory实例
                     * Cluster合并对个Invoker
                     */
                invoker = cluster.join(new StaticDirectory(u, invokers));
            }  else {
                // 没有注册中心
                invoker = cluster.join(new StaticDirectory(invokers));
            }
        }
    }

    Boolean c = check;
    if (c == null && consumer != null) {
        c = consumer.isCheck();
    }
    if (c == null) {
        c = true; // default true
    }
    if (c && ! invoker.isAvailable()) {
        throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
    }
    if (logger.isInfoEnabled()) {
        logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
    }
    // 创建服务代理
    return (T) proxyFactory.getProxy(invoker);
}
```

这里代码比较长，不全部进行分析，只看两个最重要的点：

- 创建Invoker的过程以及Cluster的合并Invoker的过程。
- proxyFactory获取代理的过程。

## 创建Invoker

Invoker的重要性不再多说，在服务提供方也会有Invoker，服务提供方的Invoker用于调用真实服务的实现类。而消费者的Invoker则是用来执行远程调用的。我们看`refprotocol.refer(interfaceClass, url)`这段代码，获取Invoker实例。这里的refprotocol是动态生成的Protocol$Adaptive，生成的代码不再贴出来，之前的文章里面有多次出现。

此时我们url是类似`registry://`开头的，所以此时refprotocol的refer其实是RegistryProtocol的refer方法：

```java
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    // 从url中得到protocol，我们这里是zookeeper，然后删除url中的registry属性
    url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
    /**
         * 获取Registry的实现类，由于我们配置的注册中心是zookeeper
         * 所以这里我们获取到的实现类是ZookeeperRegistry
         */
    Registry registry = registryFactory.getRegistry(url);
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }

    // group="a,b" or group="*"
    // 配置了group
    Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
    String group = qs.get(Constants.GROUP_KEY);
    if (group != null && group.length() > 0 ) {
        if ( ( Constants.COMMA_SPLIT_PATTERN.split( group ) ).length > 1
            || "*".equals( group ) ) {
            return doRefer( getMergeableCluster(), registry, type, url );
        }
    }
    // 引用服务
    return doRefer(cluster, registry, type, url);
}
```

这里也是先获取注册中心实现类ZookeeperRegistry，如果配置了group就处理group的逻辑，最后doRefer进行引用服务。

继续看doRefer方法：

```java
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
    /**
         * 实例化一个RegistryDirectory，Directory类似一个List，
         * 内部持有多个Invoker的列表。
         * RegistryDirectory是动态的，他实现了监听接口，里面的Invoker是可以变化的。
         * 还有一个StaticDirectory是静态的，他内部的Invoker是不可变的。
         */
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    directory.setRegistry(registry);
    directory.setProtocol(protocol);
    // 获取消费者服务的URL，以consumer://开头
    URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, NetUtils.getLocalHost(), 0, type.getName(), directory.getUrl().getParameters());
    if (! Constants.ANY_VALUE.equals(url.getServiceInterface())
        && url.getParameter(Constants.REGISTER_KEY, true)) {
        // 注册到注册中心去
        registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                                                     Constants.CHECK_KEY, String.valueOf(false)));
    }
    // 订阅providers、configurators、routers等结点
    directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY, 
                                                  Constants.PROVIDERS_CATEGORY 
                                                  + "," + Constants.CONFIGURATORS_CATEGORY 
                                                  + "," + Constants.ROUTERS_CATEGORY));
    // 一个注册中心可能有多个服务提供者，这里需要将多个提供者合并为一个Invoker
    return cluster.join(directory);
}
```

大概步骤如下：

- 实例化一个RegistryDirectory，RegistryDirectory是一个动态的持有Invoker列表的类，实现了监听器接口，Invoker可以变化。
- 获取服务消费者的URL并注册到注册中心去。
- 订阅providers、configurators、routers等结点。
- 使用Cluster合并多个提供者为一个Invoker。

先看下注册到注册中心这一步，这一步和服务提供者一样，这里registry是ZookeeperRegistry，继承了FailbackRegistry，FailbackRegistry继承了AbstractRegistry。就是将服务消费者也写入zookeeper中，具体的可以参考下服务提供者启动那篇写的，不再重复说明。

继续看合并多个服务提供者的操作，这里cluster是个自适应的Cluster，默认是FailoverCluster，但是外面会有个MockClusterWrapper进行包装。我们进入FailoverCluster的join方法看下：

```java
public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
    return new FailoverClusterInvoker<T>(directory);
}
```

并没有什么特殊逻辑，只是实例化了一个FailoverClusterInvoker，这其实是一个Invoker，但是是个失败会重试的Invoker。

我们继续往下看，这里就直接将这个FailoverClusterInvoker返回了，另外我们在提供者初始化那篇说过，protocol外面会有两层的包装，也就是ProtocolFilterWrapper和ProtocolListenerWrapper，所以这里也会经过这两个包装类处理，但是目前我们还是注册中心服务的引用，所以此时没有做其他逻辑处理。就只能跟着代码继续往下看了。

## 创建代理

上面创建了Invoker之后，就接着来到`return (T) proxyFactory.getProxy(invoker)`这一步进行代理的创建了。proxyFactory是自适应的ProxyFactory实现，并且我们没有配置，默认使用JavassistProxyFactory，并且外面有一层StubProxyFactoryWrapper包装。我们先看JavassistProxyFactory的getProxy方法：

```java
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}
```

这里Proxy.getProxy是dubbo自己实现的，然后进newInstance创建代理实例。具体的方法还没进行分析。创建完代理返回后就算流程结束了。

上面只是创建Invoker和代理的过程，引用服务的过程还没有看到，接下来就看看服务引用的过程。

# 服务引用

服务引用，我们这里说的是dubbo协议的，所以需要分析DubboProtocol的refer方法：

```java
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    // create rpc invoker.
    /**
         * 创建DubboInvoker实例
         * 先调用getClients方法服务连接
         * 
         * serviceType是接口，比如me.cxis.dubbo.service.HelloService
         * url就是dubbo://的url
         * invokers就是Invoker的一个Set
         */
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);
    return invoker;
}
```

继续getClients：

```java
private ExchangeClient[] getClients(URL url){
    // 是否共享连接
    boolean service_share_connect = false;
    int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
    // 如果connections不配置，则共享连接，否则每服务每连接
    if (connections == 0){
        service_share_connect = true;
        connections = 1;
    }

    ExchangeClient[] clients = new ExchangeClient[connections];
    for (int i = 0; i < clients.length; i++) {
        if (service_share_connect){
            clients[i] = getSharedClient(url);
        } else {
            clients[i] = initClient(url);
        }
    }
    return clients;
}
```

我们没有配置connections，则使用共享连接，getSharedCliet：

```java
private ExchangeClient getSharedClient(URL url){
    // xxx.xxx.xxx.xxx:20080
    String key = url.getAddress();
    // 引用的Client缓存
    ReferenceCountExchangeClient client = referenceClientMap.get(key);
    if ( client != null ){
        if ( !client.isClosed()){
            client.incrementAndGetCount();
            return client;
        } else {
            //                logger.warn(new IllegalStateException("client is closed,but stay in clientmap .client :"+ client));
            referenceClientMap.remove(key);
        }
    }
    // 创建新连接
    ExchangeClient exchagneclient = initClient(url);

    // 将新连接封装一下，返回
    client = new ReferenceCountExchangeClient(exchagneclient, ghostClientMap);
    referenceClientMap.put(key, client);
    ghostClientMap.remove(key);
    return client; 
}
```

创建新连接代码：

```java
private ExchangeClient initClient(URL url) {
        
    // client type setting.
    // 默认netty
    String str = url.getParameter(Constants.CLIENT_KEY, url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_CLIENT));

    // dubbo版本
    String version = url.getParameter(Constants.DUBBO_VERSION_KEY);
    boolean compatible = (version != null && version.startsWith("1.0."));
    url = url.addParameter(Constants.CODEC_KEY, Version.isCompatibleVersion() && compatible ? COMPATIBLE_CODEC_NAME : DubboCodec.NAME);
    //默认开启heartbeat
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));

    // BIO存在严重性能问题，暂时不允许使用
    if (str != null && str.length() > 0 && ! ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
        throw new RpcException("Unsupported client type: " + str + "," +
                               " supported client type is " + StringUtils.join(ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
    }

    ExchangeClient client ;
    try {
        //设置连接应该是lazy的 
        if (url.getParameter(Constants.LAZY_CONNECT_KEY, false)){
            client = new LazyConnectExchangeClient(url ,requestHandler);
        } else {
            // 连接client
            client = Exchangers.connect(url ,requestHandler);
        }
    } catch (RemotingException e) {
        throw new RpcException("Fail to create remoting client for service(" + url
                               + "): " + e.getMessage(), e);
    }
    return client;
}
```

`Exchangers.connect()`开始连接服务，继续看代码：

```java
public static ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handler == null) {
        throw new IllegalArgumentException("handler == null");
    }
    url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
    return getExchanger(url).connect(url, handler);
}
```

这里getExchanger获取到的是默认的HeaderExchanger，到connect中看下：

```java
public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
    return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
}
```

这里面先实例化一个HeaderExchangeHandler，然后实例化一个DecodeHandler，再调用Transporters的connect方法，最后实例化成一个HeaderExchangeClient返回。

其实这面往下就跟服务提供者导出服务差不多了，实例化NettyClient，连接之类的。

DubboProtocol外面还包装着ProtocolListenerWrapper和ProtocolFilterWrapper两个类，DubboProtocol的refer成功后，就会经过这两个包装类的处理，这两个在服务提供者那里也说过，不在多说。

我们来简单说下服务提供者引用服务的过程：

- Spring容器遇到dubbo的service标签，调用DubboBeanDefinitionParser解析标签。
- dubbo的bean实例化后，根据时机在afterPropertiesSet或者在getBean的时候开始引用。
- 创建Invoker实例。
- 创建代理类。
- 同时也创建NettyClient建立连接。