首先还是Spring碰到dubbo的标签之后，会使用parseCustomElement解析dubbo标签，使用的解析器是dubbo的DubboBeanDefinitionParser，解析完成之后返回BeanDefinition给Spring管理。

服务消费者端对应的是ReferenceBean，实现了ApplicationContextAware接口，Spring会在Bean的实例化那一步回调setApplicationContext方法。也实现了InitializingBean接口，接着会回调afterPropertySet方法。还实现了FactoryBean接口，实现FactoryBean可以在后期获取bean的时候做一些操作，dubbo在这个时候做初始化。另外ReferenceBean还实现了DisposableBean，会在bean销毁的时候调用destory方法。

消费者的初始化是在ReferenceBean的init方法中执行，分为两种情况：

- reference标签中没有配置init属性，此时是延迟初始化的，也就是只有等到bean引用被注入到其他Bean中，或者调用getBean获取这个Bean的时候，才会初始化。比如在这里的例子里reference没有配置init属性，只有等到`HelloService helloService = (HelloService) applicationContext.getBean("helloService");`这句getBean的时候，才会开始调用init方法进行初始化。
- 另外一种情况是立即初始化，即是如果reference标签中init属性配置为true，会立即进行初始化（也就是上面说到的实现了FactoryBean接口）。

这里以没有配置init的reference为例，只要不注入bean或者不调用getBean获取bean的时候，就不会被初始化。`HelloService helloService = (HelloService) applicationContext.getBean("helloService");`

另外在ReferenceBean这个类在Spring中初始化的时候，有几个静态变量会被初始化：

```
private static final Protocol refprotocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();

private static final Cluster cluster = ExtensionLoader.getExtensionLoader(Cluster.class).getAdaptiveExtension();

private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
```
这几个变量的初始化是根据dubbo的SPI扩展机制动态生成的代码：

refprotocol：

```
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {
  public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws java.lang.Class {
    if (arg1 == null) throw new IllegalArgumentException("url == null");

    com.alibaba.dubbo.common.URL url = arg1;
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );

    if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
    
    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
    
    return extension.refer(arg0, arg1);
  }
  
  public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.Invoker {
    if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
    
    if (arg0.getUrl() == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();
    
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
    
    if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
    
    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
    
    return extension.export(arg0);
  }
  
  public void destroy() {
  	throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
  }
  
  public int getDefaultPort() {
  	throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
  }
}
```

cluster：

```
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class Cluster$Adpative implements com.alibaba.dubbo.rpc.cluster.Cluster {

  public com.alibaba.dubbo.rpc.Invoker join(com.alibaba.dubbo.rpc.cluster.Directory arg0) throws com.alibaba.dubbo.rpc.cluster.Directory {
    if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.cluster.Directory argument == null");
    
    if (arg0.getUrl() == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.cluster.Directory argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();
    
    String extName = url.getParameter("cluster", "failover");
    if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.cluster.Cluster) name from url(" + url.toString() + ") use keys([cluster])");
    
    com.alibaba.dubbo.rpc.cluster.Cluster extension = (com.alibaba.dubbo.rpc.cluster.Cluster)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.cluster.Cluster.class).getExtension(extName);
    
    return extension.join(arg0);
  }
}
```

proxyFactory：

```
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class ProxyFactory$Adpative implements com.alibaba.dubbo.rpc.ProxyFactory {

  public java.lang.Object getProxy(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.Invoker {
    if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
    
    if (arg0.getUrl() == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();
    
    String extName = url.getParameter("proxy", "javassist");
    if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
    
    com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
    
    return extension.getProxy(arg0);
  }
  
  public com.alibaba.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, com.alibaba.dubbo.common.URL arg2) throws java.lang.Object {
    if (arg2 == null) throw new IllegalArgumentException("url == null");
    
    com.alibaba.dubbo.common.URL url = arg2;
    String extName = url.getParameter("proxy", "javassist");
    
    if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
    
    com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
    
    return extension.getInvoker(arg0, arg1, arg2);
  }
}
```

初始化的入口在ReferenceConfig的get()方法：

```
public synchronized T get() {
  if (destroyed){
  	throw new IllegalStateException("Already destroyed!");
  }
  if (ref == null) {
  	init();
  }
  return ref;
}
```
init()方法会先初始化所有的配置信息，然后调用`ref = createProxy(map);`。消费者最终得到的是服务的代理。初始化主要做的事情就是引用对应的远程服务，大概的步骤：

- 监听注册中心
- 连接服务提供者端进行服务引用
- 创建服务代理并返回

文档上关于Zookeeper作为注册中心时，服务消费者启动时要做的事情有：

> 订阅/dubbo/com.foo.BarService/providers目录下的提供者URL地址。
> 并向/dubbo/com.foo.BarService/consumers目录下写入自己的URL地址。

init()中createProxy方法：

```
private T createProxy(Map<String, String> map) {
	//先判断是否是本地服务引用injvm
    //判断是否是点对点直连
    //判断是否是通过注册中心连接
    //然后是服务的引用
    //这里url为registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=dubbo-consumer&dubbo=2.5.3&pid=12272&
    //refer=application%3Ddubbo-consumer%26dubbo%3D2.5.3%26interface%3Ddubbo.common.hello.service.HelloService%26
    //methods%3DsayHello%26pid%3D12272%26side%3Dconsumer%26timeout%3D100000%26timestamp%3D1489318676447&
    //registry=zookeeper&timestamp=1489318676641
    //引用远程服务由Protocol的实现来处理
    refprotocol.refer(interfaceClass, url);
    //最后返回服务代理
     return (T) proxyFactory.getProxy(invoker);
}
```
这里refprotocol是上面生成的代码，会根据协议不同选择不同的Protocol协议。

对于服务引用`refprotocol.refer(interfaceClass, url)`会首先进入ProtocolListenerWrapper的refer方法，然后在进入ProtocolFilterWrapper的refer方法，然后再进入RegistryProtocol的refer方法，这里的url协议是registry，所以上面两个Wrapper中不做处理，直接进入了RegistryProtocol，看下RegistryProtocol中：

```
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
	//这里获得的url是zookeeper://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=dubbo-consumer&dubbo=2.5.3&pid=12272&
    //refer=application%3Ddubbo-consumer%26dubbo%3D2.5.3%26interface%3Ddubbo.common.hello.service.HelloService%26
    //methods%3DsayHello%26pid%3D12272%26side%3Dconsumer%26timeout%3D100000%26
    //timestamp%3D1489318676447&timestamp=1489318676641
    url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
    //根据url获取Registry对象
    //先连接注册中心，把消费者注册到注册中心
    Registry registry = registryFactory.getRegistry(url);
    //判断引用是否是注册中心RegistryService，如果是直接返回刚得到的注册中心服务
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }
	//以下是普通服务，需要进入注册中心和集群下面的逻辑
    // group="a,b" or group="*"
    //获取ref的各种属性
    Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
    //获取分组属性
    String group = qs.get(Constants.GROUP_KEY);
    //先判断引用服务是否需要合并不同实现的返回结果
    if (group != null && group.length() > 0 ) {
        if ( ( Constants.COMMA_SPLIT_PATTERN.split( group ) ).length > 1
                || "*".equals( group ) ) {
                //使用默认的分组聚合集群策略
            return doRefer( getMergeableCluster(), registry, type, url );
        }
    }
    //选择配置的集群策略（cluster="failback"）或者默认策略
    return doRefer(cluster, registry, type, url);
}
```

连接注册中心`Registry registry = registryFactory.getRegistry(url);`：

```
public Registry getRegistry(URL url) {
	//这里url是zookeeper://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=dubbo-consumer&dubbo=2.5.3&
    //interface=com.alibaba.dubbo.registry.RegistryService&pid=12272&timestamp=1489318676641
    url = url.setPath(RegistryService.class.getName())
            .addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
            .removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
    //这里key是zookeeper://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService
    String key = url.toServiceString();
    // 锁定注册中心获取过程，保证注册中心单一实例
    LOCK.lock();
    try {
        Registry registry = REGISTRIES.get(key);
        if (registry != null) {
            return registry;
        }
        //这里用的是ZookeeperRegistryFactory
        //返回的Registry中封装了已经连接到Zookeeper的zkClient实例
        registry = createRegistry(url);
        if (registry == null) {
            throw new IllegalStateException("Can not create registry " + url);
        }
        //放到缓存中
        REGISTRIES.put(key, registry);
        return registry;
    } finally {
        // 释放锁
        LOCK.unlock();
    }
}
```

ZookeeperRegistryFactory的createRegistry方法：

```
public Registry createRegistry(URL url) {
	//直接返回一个新的ZookeeperRegistry实例
    //这里的zookeeperTransporter代码在下面，动态生成的适配类
    return new ZookeeperRegistry(url, zookeeperTransporter);
}
```

zookeeperTransporter代码：

```
package com.alibaba.dubbo.remoting.zookeeper;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class ZookeeperTransporter$Adpative implements com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter {
    public com.alibaba.dubbo.remoting.zookeeper.ZookeeperClient connect(com.alibaba.dubbo.common.URL arg0) {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        
        com.alibaba.dubbo.common.URL url = arg0;
        String extName = url.getParameter("client", url.getParameter("transporter", "zkclient"));
        
        if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter) name from url(" + url.toString() + ") use keys([client, transporter])");
        
        com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter extension = (com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter.class).getExtension(extName);
        
        return extension.connect(arg0);
    }
}
```
上面代码中可以看到，如果我们没有指定Zookeeper的client属性，默认使用zkClient，所以上面的zookeeperTransporter是ZkclientZookeeperTransporter。

继续看`new ZookeeperRegistry(url, zookeeperTransporter);`：

```
public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
	//这里会先经过AbstractRegistry的处理，然后经过FailbackRegistry的处理（解释在下面）
    super(url);
    if (url.isAnyHost()) {
        throw new IllegalStateException("registry address == null");
    }
    //服务分组
    String group = url.getParameter(Constants.GROUP_KEY, DEFAULT_ROOT);
    if (! group.startsWith(Constants.PATH_SEPARATOR)) {
        group = Constants.PATH_SEPARATOR + group;
    }
    this.root = group;
    //ZkclientZookeeperTransporter的connect方法
    //直接返回一个ZkclientZookeeperClient实例
    //具体的步骤是，new一个ZkClient实例，然后订阅了一个状态变化的监听器
    zkClient = zookeeperTransporter.connect(url);
    //添加一个状态改变的监听器
    zkClient.addStateListener(new StateListener() {
        public void stateChanged(int state) {
            if (state == RECONNECTED) {
                try {
                    recover();
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                }
            }
        }
    });
}
```

AbstractRegistry的处理：

```
public AbstractRegistry(URL url) {
	//设置registryUrl
    setUrl(url);
    // 启动文件保存定时器
    syncSaveFile = url.getParameter(Constants.REGISTRY_FILESAVE_SYNC_KEY, false);
    //会先去用户主目录下的.dubbo目录下加载缓存注册中心的缓存文件比如：dubbo-registry-127.0.0.1.cache
    String filename = url.getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/.dubbo/dubbo-registry-" + url.getHost() + ".cache");
    File file = null;
    if (ConfigUtils.isNotEmpty(filename)) {
        file = new File(filename);
        if(! file.exists() && file.getParentFile() != null && ! file.getParentFile().exists()){
            if(! file.getParentFile().mkdirs()){
                throw new IllegalArgumentException("Invalid registry store file " + file + ", cause: Failed to create directory " + file.getParentFile() + "!");
            }
        }
    }
    this.file = file;
    //缓存文件存在的话就把文件读进内存中
    loadProperties();
    //先获取backup url
    //然后notify（暂没理解）
    notify(url.getBackupUrls());
}
```

接着是FailbackRegistry的处理：

```
public FailbackRegistry(URL url) {
    super(url);
    int retryPeriod = url.getParameter(Constants.REGISTRY_RETRY_PERIOD_KEY, Constants.DEFAULT_REGISTRY_RETRY_PERIOD);
    this.retryFuture = retryExecutor.scheduleWithFixedDelay(new Runnable() {
        public void run() {
            // 检测并连接注册中心
            try {
                retry();
            } catch (Throwable t) { // 防御性容错
                logger.error("Unexpected error occur at failed retry, cause: " + t.getMessage(), t);
            }
        }
    }, retryPeriod, retryPeriod, TimeUnit.MILLISECONDS);
}
```
这里会启动一个新的定时线程，主要是有连接失败的话，会进行重试连接retry();，启动完之后返回ZookeeperRegistry中继续处理。


继续看ref方法中最后一步，服务的引用，返回的是一个Invoker，`return doRefer(cluster, registry, type, url)；`

```
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
	//初始化Directory
    //组装Directory，可以看成一个消费端的List，可以随着注册中心的消息推送而动态的变化服务的Invoker
    //封装了所有服务真正引用逻辑，覆盖配置，路由规则等逻辑
    //初始化时只需要向注册中心发起订阅请求，其他逻辑均是异步处理，包括服务的引用等
    //缓存接口所有的提供者端Invoker以及注册中心接口相关的配置等
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    directory.setRegistry(registry);
    directory.setProtocol(protocol);
    //此处的subscribeUrl为consumer://192.168.1.100/dubbo.common.hello.service.HelloService?application=dubbo-consumer&dubbo=2.5.3&
    //interface=dubbo.common.hello.service.HelloService&methods=sayHello&pid=16409&
    //side=consumer&timeout=100000&timestamp=1489322133987
    URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, NetUtils.getLocalHost(), 0, type.getName(), directory.getUrl().getParameters());
    if (! Constants.ANY_VALUE.equals(url.getServiceInterface())
            && url.getParameter(Constants.REGISTER_KEY, true)) {
            //到注册中心注册服务
            //此处regist是上面一步获得的registry，即是ZookeeperRegistry，包含zkClient的实例
            //会先经过AbstractRegistry的处理，然后经过FailbackRegistry的处理（解析在下面）
        registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                Constants.CHECK_KEY, String.valueOf(false)));
    }
    //订阅服务
    //有服务提供的时候，注册中心会推送服务消息给消费者，消费者再进行服务的引用。
    directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY, 
            Constants.PROVIDERS_CATEGORY 
            + "," + Constants.CONFIGURATORS_CATEGORY 
            + "," + Constants.ROUTERS_CATEGORY));
    //服务的引用与变更全部由Directory异步完成
    //集群策略会将Directory伪装成一个Invoker返回
    return cluster.join(directory);
}
```

注册中心接收到消费者发送的订阅请求后，会根据提供者注册服务的列表，推送服务消息给消费者。消费者端接收到注册中心发来的提供者列表后，进行服务的引用。触发Directory监听器的可以是订阅请求，覆盖策略消息，路由策略消息。

AbstractRegistry的register方法：

```
public void register(URL url) {
	//此时url是consumer://192.168.1.100/dubbo.common.hello.service.HelloService?application=dubbo-consumer&
    //category=consumers&check=false&dubbo=2.5.3&interface=dubbo.common.hello.service.HelloService&methods=sayHello
    //&pid=16409&side=consumer&timeout=100000&timestamp=1489322133987
    if (url == null) {
        throw new IllegalArgumentException("register url == null");
    }
    if (logger.isInfoEnabled()){
        logger.info("Register: " + url);
    }
    registered.add(url);
}
```
上面只是把url添加到registered这个set中。

接着看FailbackRegistry的register方法：

```
public void register(URL url) {
    super.register(url);
    failedRegistered.remove(url);
    failedUnregistered.remove(url);
    try {
        // 向服务器端发送注册请求
        //这里调用的是ZookeeperRegistry中的doRegister方法
        doRegister(url);
    } catch (Exception e) {
        Throwable t = e;

        // 如果开启了启动时检测，则直接抛出异常
        boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                && url.getParameter(Constants.CHECK_KEY, true)
                && ! Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
        boolean skipFailback = t instanceof SkipFailbackWrapperException;
        if (check || skipFailback) {
            if(skipFailback) {
                t = t.getCause();
            }
            throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
        } else {
            logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
        }

        // 将失败的注册请求记录到失败列表，定时重试
        failedRegistered.add(url);
    }
}
```

接着看下doRegister(url);方法，向服务器端发送注册请求，在ZookeeperRegistry中：

```
protected void doRegister(URL url) {
    try {
    	//直接调用create，在AbstractZookeeperClient类中
        zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
    } catch (Throwable e) {
        throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

zkClient.create()方法：

```
//path为/dubbo/dubbo.common.hello.service.HelloService/consumers/consumer%3A%2F%2F192.168.1.100%2F
//dubbo.common.hello.service.HelloService%3Fapplication%3Ddubbo-consumer%26category%3Dconsumers%26check%3Dfalse%26dubbo%3D2.5.3%26interface%3D
//dubbo.common.hello.service.HelloService%26methods%3DsayHello%26pid%3D28819%26
//side%3Dconsumer%26timeout%3D100000%26timestamp%3D1489332839677
public void create(String path, boolean ephemeral) {
    int i = path.lastIndexOf('/');
    if (i > 0) {
        create(path.substring(0, i), false);
    }
    //循环完得到的path为/dubbo
    //dynamic=false 表示该数据为持久数据，当注册方退出时，数据依然保存在注册中心
    if (ephemeral) {
    	//创建临时的节点，/dubbo
        createEphemeral(path);
    } else {
    	//创建持久的节点，/dubbo
        createPersistent(path);
    }
}
```

注册到注册中心之后，是监听注册中心，directory.subscribe()，会先经过AbstractRegistry的处理，然后是FailbackRegistry的处理。

在AbstractRegistry中：

```
//此时url为consumer://192.168.1.100/dubbo.common.hello.service.HelloService?application=dubbo-consumer&
//category=providers,configurators,routers&dubbo=2.5.3&interface=dubbo.common.hello.service.HelloService&methods=
//sayHello&pid=28819&side=consumer&timeout=100000&timestamp=1489332839677
public void subscribe(URL url, NotifyListener listener) {
	//先根据url获取已注册的监听器
    Set<NotifyListener> listeners = subscribed.get(url);
    //没有监听器，就创建，并添加进去
    if (listeners == null) {
        subscribed.putIfAbsent(url, new ConcurrentHashSet<NotifyListener>());
        listeners = subscribed.get(url);
    }
    //有监听器，直接把当前RegistryDirectory添加进去
    listeners.add(listener);
}
```

然后是FailbackRegistry中：

```
public void subscribe(URL url, NotifyListener listener) {
    super.subscribe(url, listener);
    removeFailedSubscribed(url, listener);
    try {
        // 向服务器端发送订阅请求
        doSubscribe(url, listener);
    } catch (Exception e) {...}
}
```

继续看doSubscribe(url, listener);向服务端发送订阅请求，在ZookeeperRegistry中：

```
protected void doSubscribe(final URL url, final NotifyListener listener) {
    try {
        if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {... } else {
            List<URL> urls = new ArrayList<URL>();
            for (String path : toCategoriesPath(url)) {
                ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                if (listeners == null) {
                    zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                    listeners = zkListeners.get(url);
                }
                ChildListener zkListener = listeners.get(listener);
                if (zkListener == null) {
                    listeners.putIfAbsent(listener, new ChildListener() {
                        public void childChanged(String parentPath, List<String> currentChilds) {
                            ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
                        }
                    });
                    zkListener = listeners.get(listener);
                }
                //
                zkClient.create(path, false);
                List<String> children = zkClient.addChildListener(path, zkListener);
                if (children != null) {
                    urls.addAll(toUrlsWithEmpty(url, path, children));
                }
            }
            notify(url, listener, urls);
        }
    } catch (Throwable e) {
        throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

```
public void subscribe(URL url) {
	//设置消费者url
    setConsumerUrl(url);
    //这里的registry是ZookeeperRegistry
    registry.subscribe(url, this);
}
```

服务的引用DubboProtocol.refer(type,url)：

```
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    //创建一个消费者端的Invoker
    //getClients是NIO框架Client的创建
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);
    return invoker;
}
```
getClients负责NIO框架Client创建以及初始化：

```
private ExchangeClient[] getClients(URL url){
    //是否共享连接
    boolean service_share_connect = false;
    int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
    //如果connections不配置，则共享连接，否则每服务每连接
    if (connections == 0){
        service_share_connect = true;
        connections = 1;
    }

    ExchangeClient[] clients = new ExchangeClient[connections];
    for (int i = 0; i < clients.length; i++) {
        if (service_share_connect){
        	//共享型的client
            clients[i] = getSharedClient(url);
        } else {
        	//不共享的client
            clients[i] = initClient(url);
        }
    }
    return clients;
}
```
共享型的client，消费者引用同一提供者的服务时，使用同一个Client来提高通信效率。

非共享型的client，initClient。封装了服务引用中remote层初始化的所有逻辑，与服务提供者端类似，就是Client，Handler，Channel的创建和装饰的过程。

