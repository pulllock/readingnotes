dubbo暴露服务有两种情况，一种是设置了延迟暴露（比如delay="5000"），另外一种是没有设置延迟暴露或者延迟设置为-1（delay="-1"）：

- 设置了延迟暴露，dubbo在Spring实例化bean（initializeBean）的时候会对实现了InitializingBean的类进行回调，回调方法是afterPropertySet()，如果设置了延迟暴露，dubbo在这个方法中进行服务的发布。
- 没有设置延迟或者延迟为-1，dubbo会在Spring实例化完bean之后，在刷新容器最后一步发布ContextRefreshEvent事件的时候，通知实现了ApplicationListener的类进行回调onApplicationEvent，dubbo会在这个方法中发布服务。

但是不管延迟与否，都是使用ServiceConfig的export()方法进行服务的暴露。使用export初始化的时候会将Bean对象转换成URL格式，所有Bean属性转换成URL的参数。

# 过程
以没有设置延迟暴露熟属性的过程为例。

## 简易的暴露流程

1. 首先将服务的实现封装成一个Invoker，Invoker中封装了服务的实现类。
2. 将Invoker封装成Exporter，并缓存起来，缓存里使用Invoker的url作为key。
3. 服务端Server启动，监听端口。（请求来到时，根据请求信息生成key，到缓存查找Exporter，就找到了Invoker，就可以完成调用。）

## Spring容器初始化调用
当Spring容器实例化bean完成，走到最后一步发布ContextRefreshEvent事件的时候，ServiceBean会执行onApplicationEvent方法，该方法调用ServiceConfig的export方法。

ServiceConfig初始化的时候，会先初始化静态变量protocol和proxyFactory，这两个变量初始化的结果是通过dubbo的spi扩展机制得到的。

生成的protocol实例是：

```
package com.alibaba.dubbo.rpc;
import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {
    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws java.lang.Class {
        if (arg1 == null) 
            throw new IllegalArgumentException("url == null");

        com.alibaba.dubbo.common.URL url = arg1;
        String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
        if(extName == null) 
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");

        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);

        return extension.refer(arg0, arg1);
    }

    public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.Invoker {
        if (arg0 == null) 
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");

        if (arg0.getUrl() == null) 
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();
        //根据URL配置信息获取Protocol协议，默认是dubbo
        String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
        if(extName == null) 
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
            //根据协议名，获取Protocol的实现
            //获得Protocol的实现过程中，会对Protocol先进行依赖注入，然后进行Wrapper包装，最后返回被修改过的Protocol
            //包装经过了ProtocolFilterWrapper，ProtocolListenerWrapper，RegistryProtocol
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

生成的proxyFactory实例：

```
package com.alibaba.dubbo.rpc;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class ProxyFactory$Adpative implements com.alibaba.dubbo.rpc.ProxyFactory {
    public com.alibaba.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, com.alibaba.dubbo.common.URL arg2) throws java.lang.Object {
        if (arg2 == null) 
            throw new IllegalArgumentException("url == null");

        com.alibaba.dubbo.common.URL url = arg2;
        String extName = url.getParameter("proxy", "javassist");
        if(extName == null) 
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");

        com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);

        return extension.getInvoker(arg0, arg1, arg2);
    }

    public java.lang.Object getProxy(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.Invoker {
        if (arg0 == null) 
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");

       if (arg0.getUrl() == null) 
        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();

        String extName = url.getParameter("proxy", "javassist");
        if(extName == null) 
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");

        com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);

        return extension.getProxy(arg0);
    }
}
```
生成的代码中可以看到，默认的Protocol实现是dubbo，默认的proxy是javassist。

## ServiceConfig的export

### export的步骤简介

1. 首先会检查各种配置信息，填充各种属性，总之就是保证我在开始暴露服务之前，所有的东西都准备好了，并且是正确的。
2. 加载所有的注册中心，因为我们暴露服务需要注册到注册中心中去。
3. 根据配置的所有协议和注册中心url分别进行导出。
4. 进行导出的时候，又是一波属性的获取设置检查等操作。
5. 如果配置的不是remote，则做本地导出。
6. 如果配置的不是local，则暴露为远程服务。
7. 不管是本地还是远程服务暴露，首先都会获取Invoker。
8. 获取完Invoker之后，转换成对外的Exporter，缓存起来。

export方法先判断是否需要延迟暴露（这里我们使用的是不延迟暴露），然后执行doExport方法。

doExport方法先执行一系列的检查方法，然后调用doExportUrls方法。检查方法会检测dubbo的配置是否在Spring配置文件中声明，没有的话读取properties文件初始化。

doExportUrls方法先调用loadRegistries获取所有的注册中心url，然后遍历调用doExportUrlsFor1Protocol方法。对于在标签中指定了registry属性的Bean，会在加载BeanDefinition的时候就加载了注册中心。
	
获取注册中心url，会把注册的信息都放在一个URL对象中，一个URL内容如下：
    
```
registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=dubbo-provider&application.version=1.0&dubbo=2.5.3&environment=product&organization=china&owner=cheng.xi&pid=2939&registry=zookeeper&timestamp=1488898049284
```
    
doExportUrlsFor1Protocol根据不同的协议将服务以URL形式暴露。如果scope配置为none则不暴露，如果服务未配置成remote，则本地暴露exportLocal，如果未配置成local，则注册服务registryProcotol。
	
 这个方法中会把上面的注册信息的URL转换成协议信息URL：
    
```
dubbo://192.168.1.100:20880/dubbo.common.hello.service.HelloService?anyhost=true&application=dubbo-provider&application.version=1.0&delay=5000&dubbo=2.5.3&environment=product&interface=dubbo.common.hello.service.HelloService&methods=sayHello&organization=china&owner=cheng.xi&pid=2939&side=provider&timestamp=1488898464953
```
接下来会调用进行服务的发布：
    
```
//根据服务具体实现，实现接口，以及registryUrl通过ProxyFactory将HelloServiceImpl封装成一个本地执行的Invoker
//invoker是对具体实现的一种代理。
//这里proxyFactory是上面列出的生成的代码
 Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
 //使用Protocol将invoker导出成一个Exporter
 //暴露封装服务invoker
 //调用Protocol生成的适配类的export方法
 //这里的protocol是上面列出的生成的代码
 Exporter<?> exporter = protocol.export(invoker);
```
**关于Invoker，Exporter等的解释参见下面的内容。**
   
这里还是以dubbo协议为例子，对应的Protocol为DubboProtocol，使用SPI机制获取到DubboProtocol的过程中，就已经进行了ProtocolFilterWrapper、ProtocolListenerWrapper、RegistryProtocol等的包装。

## 获取Invoker，服务实现类转换成Invoker

```
Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
```
这行代码中包含服务实现类转换成Invoker的过程，其中proxyFactory是上面列出的动态生成的代码，其中getInvoker的代码为（做了精简，把包都去掉了）：

```
public Invoker getInvoker(Object arg0, Class arg1, URL arg2) throws Object {
    if (arg2 == null)  throw new IllegalArgumentException("url == null");
	//传进来的url是dubbo://192.168.110.197:20880/dubbo.common.hello.service.HelloService?anyhost=true&application=dubbo-provider
    //&application.version=1.0&dubbo=2.5.3&environment=product&interface=dubbo.common.hello.service.HelloService&methods=sayHello&organization=china&owner=cheng.xi
    //&pid=28191&side=provider&timestamp=1489027396094
    URL url = arg2;
    //没有proxy参数配置，默认使用javassist
    String extName = url.getParameter("proxy", "javassist");
    if(extName == null)  throw new IllegalStateException("Fail to get extension(ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
	//这一步就使用javassist来获取ProxyFactory的实现类JavassistProxyFactory
    ProxyFactory extension = (ProxyFactory)ExtensionLoader.getExtensionLoader(ProxyFactory.class).getExtension(extName);
	//JavassistProxyFactory的getInvoker方法
    return extension.getInvoker(arg0, arg1, arg2);
}
```

JavassistProxyFactory的getInvoker方法：

```
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    // TODO Wrapper类不能正确处理带$的类名
    //第一步封装一个Wrapper类
    //该类是手动生成的
    //如果类是以$开头，就使用接口类型获取，其他的使用实现类获取
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName, 
                                  Class<?>[] parameterTypes, 
                                  Object[] arguments) throws Throwable {
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```

生成wrapper类的过程：

`final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);`

首先看getWrapper方法：

```
public static Wrapper getWrapper(Class<?> c){
    while( ClassGenerator.isDynamicClass(c) ) // can not wrapper on dynamic class.
        c = c.getSuperclass();
	//Object类型的
    if( c == Object.class )
        return OBJECT_WRAPPER;
	//先去Wrapper缓存中查找
    Wrapper ret = WRAPPER_MAP.get(c);
    if( ret == null ) {
    	//缓存中不存在，生成Wrapper类，放到缓存
        ret = makeWrapper(c);
        WRAPPER_MAP.put(c,ret);
    }
    return ret;
}
```

makeWrapper方法代码不在列出，太长了。就是生成一个继承自Wrapper的类，最后的结果大概是：

```
public class Wrapper1 extends Wrapper {
    public static String[] pns;
    public static Map pts;
    public static String[] mns; // all method name array.
    public static String[] dmns;
    public static Class[] mts0;

    public String[] getPropertyNames() {
        return pns;
    }

    public boolean hasProperty(String n) {
        return pts.containsKey($1);
    }

    public Class getPropertyType(String n) {
        return (Class) pts.get($1);
    }

    public String[] getMethodNames() {
        return mns;
    }

    public String[] getDeclaredMethodNames() {
        return dmns;
    }

    public void setPropertyValue(Object o, String n, Object v) {
        dubbo.provider.hello.service.impl.HelloServiceImpl w;
        try {
            w = ((dubbo.provider.hello.service.impl.HelloServiceImpl) $1);
        } catch (Throwable e) {
            throw new IllegalArgumentException(e);
        }
        throw new com.alibaba.dubbo.common.bytecode.NoSuchPropertyException("Not found property \"" + $2 + "\" filed or setter method in class dubbo.provider.hello.service.impl.HelloServiceImpl.");
    }

    public Object getPropertyValue(Object o, String n) {
        dubbo.provider.hello.service.impl.HelloServiceImpl w;
        try {
            w = ((dubbo.provider.hello.service.impl.HelloServiceImpl) $1);
        } catch (Throwable e) {
            throw new IllegalArgumentException(e);
        }
        throw new com.alibaba.dubbo.common.bytecode.NoSuchPropertyException("Not found property \"" + $2 + "\" filed or setter method in class dubbo.provider.hello.service.impl.HelloServiceImpl.");
    }

    public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws java.lang.reflect.InvocationTargetException {
        dubbo.provider.hello.service.impl.HelloServiceImpl w;
        try {
            w = ((dubbo.provider.hello.service.impl.HelloServiceImpl) $1);
        } catch (Throwable e) {
            throw new IllegalArgumentException(e);
        }
        try {
            if ("sayHello".equals($2) && $3.length == 0) {
                w.sayHello();
                return null;
            }
        } catch (Throwable e) {
            throw new java.lang.reflect.InvocationTargetException(e);
        }
        throw new com.alibaba.dubbo.common.bytecode.NoSuchMethodException("Not found method \"" + $2 + "\" in class dubbo.provider.hello.service.impl.HelloServiceImpl.");
    }
}
```

生成完Wrapper以后，返回一个AbstractProxyInvoker实例。至此生成Invoker的步骤就完成了。接着就是暴露Invoker。

## 暴露封装着服务的Invoker

```
Exporter<?> exporter = protocol.export(invoker);
```
protocol是上面列出的动态生成的代码，会先调用ProtocolListenerWrapper，这个Wrapper负责初始化暴露和引用服务的监听器。代码如下：

```
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
	//registry类型的Invoker
    if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
        return protocol.export(invoker);
    }
    return new ListenerExporterWrapper<T>(protocol.export(invoker), 
            Collections.unmodifiableList(ExtensionLoader.getExtensionLoader(ExporterListener.class)
                    .getActivateExtension(invoker.getUrl(), Constants.EXPORTER_LISTENER_KEY)));
}
```

接着调用ProtocolFilterWrapper中的export方法，ProtocolFilterWrapper负责初始化invoker所有的Filter。代码如下：

```
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
        return protocol.export(invoker);
    }
    return protocol.export(buildInvokerChain(invoker, Constants.SERVICE_FILTER_KEY, Constants.PROVIDER));
}
```

接着就会调用RegistryProtocol的export方法，RegistryProtocol负责注册服务到注册中心和向注册中心订阅服务。代码如下：

```
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    //export invoker
    //这里就交给了具体的协议去暴露服务
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
    //registry provider
    //根据invoker中的url获取Registry实例
    //并且连接到注册中心
    //此时提供者作为消费者引用注册中心核心服务RegistryService
    final Registry registry = getRegistry(originInvoker);
    //注册到注册中心的URL
    final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
    //调用远端注册中心的register方法进行服务注册
    //若有消费者订阅此服务，则推送消息让消费者引用此服务。
    //注册中心缓存了所有提供者注册的服务以供消费者发现。
    registry.register(registedProviderUrl);
    // 订阅override数据
    // FIXME 提供者订阅时，会影响同一JVM即暴露服务，又引用同一服务的的场景，因为subscribed以服务名为缓存的key，导致订阅信息覆盖。
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    //提供者向注册中心订阅所有注册服务的覆盖配置
    //当注册中心有此服务的覆盖配置注册进来时，推送消息给提供者，重新暴露服务，这由管理页面完成。
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    //保证每次export都返回一个新的exporter实例
    //返回暴露后的Exporter给上层ServiceConfig进行缓存，便于后期撤销暴露。
    return new Exporter<T>() {
        public Invoker<T> getInvoker() {
            return exporter.getInvoker();
        }
        public void unexport() {
            try {
                exporter.unexport();
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
            try {
                registry.unregister(registedProviderUrl);
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
            try {
                overrideListeners.remove(overrideSubscribeUrl);
                registry.unsubscribe(overrideSubscribeUrl, overrideSubscribeListener);
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        }
    };
}
```

### 交给具体的协议进行服务暴露
doLocalExport(invoker)：

```
private <T> ExporterChangeableWrapper<T>  doLocalExport(final Invoker<T> originInvoker){
	//原始的invoker中的url：registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=dubbo-provider&application.version=1.0&dubbo=2.5.3
    //&environment=product&
    //export=dubbo%3A%2F%2F10.42.0.1%3A20880%2Fdubbo.common.hello.service.HelloService%3Fanyhost%3Dtrue%26application%3Ddubbo-provider%26
    //application.version%3D1.0%26dubbo%3D2.5.3%26environment%3Dproduct%26interface%3Ddubbo.common.hello.service.HelloService%26methods%3DsayHello%26
    //organization%3Dchina%26owner%3Dcheng.xi%26pid%3D7876%26side%3Dprovider%26timestamp%3D1489057305001&
    //organization=china&owner=cheng.xi&pid=7876&registry=zookeeper&timestamp=1489057304900
    //从原始的invoker中得到的key：dubbo://10.42.0.1:20880/dubbo.common.hello.service.HelloService?anyhost=true&application=dubbo-provider&
    //application.version=1.0&dubbo=2.5.3&environment=product&interface=dubbo.common.hello.service.HelloService&
    //methods=sayHello&organization=china&owner=cheng.xi&pid=7876&side=provider&timestamp=1489057305001
    String key = getCacheKey(originInvoker);
    ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
    if (exporter == null) {
        synchronized (bounds) {
            exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
            if (exporter == null) {
                final Invoker<?> invokerDelegete = new InvokerDelegete<T>(originInvoker, getProviderUrl(originInvoker));
                //此处protocol还是最上面生成的代码，调用代码中的export方法，会根据协议名选择调用具体的实现类
                //这里我们需要调用DubboProtocol的export方法
                //这里的invoker在上一步做了修改
                exporter = new ExporterChangeableWrapper<T>((Exporter<T>)protocol.export(invokerDelegete), originInvoker);
                bounds.put(key, exporter);
            }
        }
    }
    return (ExporterChangeableWrapper<T>) exporter;
}
```

下面去具体的DubboProtocol中查看调用，在调用DubboProtocol之前，还是先经过ProtocolListenerWrapper初始化监听器，再经过ProtocolFilterWrapper初始化所有的Filter，最后到了DubboProtocol的export方法，这里进行暴露服务：

```
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
	//dubbo://10.42.0.1:20880/dubbo.common.hello.service.HelloService?anyhost=true&application=dubbo-provider&
    //application.version=1.0&dubbo=2.5.3&environment=product&interface=dubbo.common.hello.service.HelloService&
    //methods=sayHello&organization=china&owner=cheng.xi&pid=7876&side=provider&timestamp=1489057305001
    URL url = invoker.getUrl();

    // export service.
    //key由serviceName，port，version，group组成
    //当nio客户端发起远程调用时，nio服务端通过此key来决定调用哪个Exporter，也就是执行的Invoker。
    String key = serviceKey(url);
    //将Invoker转换成Exporter
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    //缓存要暴露的服务，key是上面生成的
    exporterMap.put(key, exporter);

    //export an stub service for dispaching event
    Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY,Constants.DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice){
        String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0 ){
            if (logger.isWarnEnabled()){
                logger.warn(new IllegalStateException("consumer [" +url.getParameter(Constants.INTERFACE_KEY) +
                        "], has set stubproxy support event ,but no stub methods founded."));
            }
        } else {
            stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
        }
    }
	//根据URL绑定IP与端口，建立NIO框架的Server
    openServer(url);

    return exporter;
}
```
上面将Invoker转换成Exporter之后，缓存起来，接着调用openServer方法创建NIO Server：

```
private void openServer(URL url) {
    // find server.
    //key是IP:PORT
    String key = url.getAddress();
    //client 也可以暴露一个只有server可以调用的服务。
    boolean isServer = url.getParameter(Constants.IS_SERVER_KEY,true);
    if (isServer) {
    	
        ExchangeServer server = serverMap.get(key);
        //同协议的服务，第一个暴露服务的时候创建server
        if (server == null) {
            serverMap.put(key, createServer(url));
        } else {
        	//同协议的服务后来暴露服务的则使用第一次创建的同一Server
            //server支持reset,配合override功能使用
            server.reset(url);
        }
    }
}
```
同一JVM中，相同协议的服务，共享同一个Server，不同服务中只有accept、idleTimeout、threads、heartbeat参数的变化会引用Server中属性的变化，但是同JVM中，同协议的服务均是引用同一个Server。第一个服务暴露是创建Server，后来的服务暴露最多是重置个别参数。

继续看createServer方法：

```
private ExchangeServer createServer(URL url) {
    //默认开启server关闭时发送readonly事件
    url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
    //默认开启heartbeat
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
    //默认使用netty
    String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);

    if (str != null && str.length() > 0 && ! ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str))
        throw new RpcException("Unsupported server type: " + str + ", url: " + url);

    url = url.addParameter(Constants.CODEC_KEY, Version.isCompatibleVersion() ? COMPATIBLE_CODEC_NAME : DubboCodec.NAME);
    ExchangeServer server;
    try {
    	//Exchangers是门面类，里面封装的是Exchanger的逻辑。
        //Exchanger默认只有一个实现HeaderExchanger.
        //Exchanger负责数据交换和网络通信。
        //从Protocol进入Exchanger，标志着程序进入了remote层。
        server = Exchangers.bind(url, requestHandler);
    } catch (RemotingException e) {
        throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
    }
    str = url.getParameter(Constants.CLIENT_KEY);
    if (str != null && str.length() > 0) {
        Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
        if (!supportedTypes.contains(str)) {
            throw new RpcException("Unsupported client type: " + str);
        }
    }
    return server;
}
```

Exchangers.bind方法：

```
public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
    url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
    //getExchanger方法根据url获取到一个默认的实现HeaderExchanger
    //调用HeaderExchanger的bind方法
    return getExchanger(url).bind(url, handler);
}
```

HeaderExchanger的bind方法：

```
public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
	//直接返回一个HeaderExchangeServer
    //先创建一个HeaderExchangeHandler
    //再创建一个DecodeHandler
    //最后调用Transporters.bind
    return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
}
```

Transports的bind方法：

```
public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
    ChannelHandler handler;
    if (handlers.length == 1) {
        handler = handlers[0];
    } else {
        handler = new ChannelHandlerDispatcher(handlers);
    }
    //getTransporter()获取一个Adaptive的Transporter
    //然后调用bind方法（默认是NettyTransporter的bind方法）
    return getTransporter().bind(url, handler);
}
```

getTransporter()生成的Transporter的代码如下：

```
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class Transporter$Adpative implements com.alibaba.dubbo.remoting.Transporter {
    public com.alibaba.dubbo.remoting.Server bind(com.alibaba.dubbo.common.URL arg0, com.alibaba.dubbo.remoting.ChannelHandler arg1) throws com.alibaba.dubbo.common.URL {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg0;
        //Server默认使用netty
        String extName = url.getParameter("server", url.getParameter("transporter", "netty"));
        if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" + url.toString() + ") use keys([server, transporter])");
        //获取到一个NettyTransporter
        com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Transporter.class).getExtension(extName);
        //调用NettyTransporter的bind方法
        return extension.bind(arg0, arg1);
    }
    
public com.alibaba.dubbo.remoting.Client connect(com.alibaba.dubbo.common.URL arg0, com.alibaba.dubbo.remoting.ChannelHandler arg1) throws com.alibaba.dubbo.common.URL {
    if (arg0 == null) throw new IllegalArgumentException("url == null");
    com.alibaba.dubbo.common.URL url = arg0;
    
    String extName = url.getParameter("client", url.getParameter("transporter", "netty"));
    
    if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" + url.toString() + ") use keys([client, transporter])");
    
    com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Transporter.class).getExtension(extName);
    
    return extension.connect(arg0, arg1);
}
}
```

NettyTransporter的bind方法：

```
 public Server bind(URL url, ChannelHandler listener) throws RemotingException {
 	//创建一个Server
    return new NettyServer(url, listener);
}
```
newNettyServer创建一个Server，最终在AbstractServer初始化：

```
public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
    super(url, handler);
    localAddress = getUrl().toInetSocketAddress();
    String host = url.getParameter(Constants.ANYHOST_KEY, false) 
                    || NetUtils.isInvalidLocalHost(getUrl().getHost()) 
                    ? NetUtils.ANYHOST : getUrl().getHost();
    bindAddress = new InetSocketAddress(host, getUrl().getPort());
    this.accepts = url.getParameter(Constants.ACCEPTS_KEY, Constants.DEFAULT_ACCEPTS);
    this.idleTimeout = url.getParameter(Constants.IDLE_TIMEOUT_KEY, Constants.DEFAULT_IDLE_TIMEOUT);
    try {
        doOpen();
        if (logger.isInfoEnabled()) {
            logger.info("Start " + getClass().getSimpleName() + " bind " + getBindAddress() + ", export " + getLocalAddress());
        }
    } catch (Throwable t) {
        throw new RemotingException(url.toInetSocketAddress(), null, "Failed to bind " + getClass().getSimpleName() 
                                    + " on " + getLocalAddress() + ", cause: " + t.getMessage(), t);
    }
    if (handler instanceof WrappedChannelHandler ){
        executor = ((WrappedChannelHandler)handler).getExecutor();
    }
}
```
然后调用doOpen方法：

```
protected void doOpen() throws Throwable {
    NettyHelper.setNettyLoggerFactory();
    //boss线程池
    ExecutorService boss = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerBoss", true));
    //worker线程池
    ExecutorService worker = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerWorker", true));
    //ChannelFactory，没有指定工作者线程数量，就使用cpu+1
    ChannelFactory channelFactory = new NioServerSocketChannelFactory(boss, worker, getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS));
    bootstrap = new ServerBootstrap(channelFactory);

    final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
    channels = nettyHandler.getChannels();
    // https://issues.jboss.org/browse/NETTY-365
    // https://issues.jboss.org/browse/NETTY-379
    // final Timer timer = new HashedWheelTimer(new NamedThreadFactory("NettyIdleTimer", true));
    bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
        public ChannelPipeline getPipeline() {
            NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec() ,getUrl(), NettyServer.this);
            ChannelPipeline pipeline = Channels.pipeline();
            /*int idleTimeout = getIdleTimeout();
            if (idleTimeout > 10000) {
                pipeline.addLast("timer", new IdleStateHandler(timer, idleTimeout / 1000, 0, 0));
            }*/
            pipeline.addLast("decoder", adapter.getDecoder());
            pipeline.addLast("encoder", adapter.getEncoder());
            pipeline.addLast("handler", nettyHandler);
            return pipeline;
        }
    });
    // bind
    channel = bootstrap.bind(getBindAddress());
}
```

doOpen方法创建Netty的Server端并打开。

- NIO框架接受到消息后，先由NettyCodecAdapter解码，再由NettyHandler处理具体的业务逻辑，再由NettyCodecAdapter编码后发送。
- NettyServer既是Server又是Handler。
- HeaderExchangerServer只是Server。
- MultiMessageHandler是多消息处理Handler。
- HeartbeatHandler是处理心跳事件的Handler。
- AllChannelHandler是消息派发器，负责将请求放入线程池，并执行请求。
- DecodeHandler是编解码Handler。
- HeaderExchangerHandler是信息交换Handler，将请求转化成请求响应模式与同步转异步模式。
- RequestHandler是最后执行的Handler，会在协议层选择Exporter后选择Invoker，进而执行Filter与Invoker，最终执行请求服务实现类方法。
- Channel直接触发事件并执行Handler，Channel在有客户端连接Server的时候触发创建并封装成NettyChannel，再由HeaderExchangerHandler创建HeaderExchangerChannel，负责请求响应模式的处理。
- NettyChannel其实是个Handler，HeaderExchangerChannel是个Channel，
- 消息的序列化与反序列化工作在NettyCodecAdapter中发起完成。

当有客户端连接Server时的连接过程：

- NettyHandler.connected()
- NettyServer.connected()
- MultiMessageHandler.connected()
- HeartbeatHandler.connected()
- AllChannelHandler.connected()
- DecodeHandler.connected()
- HeaderExchangerHandler.connected()
- requestHandler.connected()
- 执行服务的onconnect事件的监听方法

### 注册到注册中心
上面讲完了交给具体的协议进行暴露，并且返回了一个Server，接下来看下一步，代码是在RegistryProtocol的export方法：

```
//此步已经分析完
final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
//得到具体的注册中心，连接注册中心，此时提供者作为消费者引用注册中心核心服务RegistryService
final Registry registry = getRegistry(originInvoker);
final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
//调用远端注册中心的register方法进行服务注册
//若有消费者订阅此服务，则推送消息让消费者引用此服务
registry.register(registedProviderUrl);
//提供者向注册中心订阅所有注册服务的覆盖配置
registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
//返回暴露后的Exporter给上层ServiceConfig进行缓存
return new Exporter<T>() {。。。}
```

getRegistry(originInvoker)方法：

```
//根据invoker的地址获取registry实例
private Registry getRegistry(final Invoker<?> originInvoker){
    URL registryUrl = originInvoker.getUrl();
    if (Constants.REGISTRY_PROTOCOL.equals(registryUrl.getProtocol())) {
        String protocol = registryUrl.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_DIRECTORY);
        registryUrl = registryUrl.setProtocol(protocol).removeParameter(Constants.REGISTRY_KEY);
    }
    return registryFactory.getRegistry(registryUrl);
}
```
这里的registryFactory是动态生成的代码，如下：

```
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class RegistryFactory$Adpative implements com.alibaba.dubbo.registry.RegistryFactory {
    public com.alibaba.dubbo.registry.Registry getRegistry(com.alibaba.dubbo.common.URL arg0) {
    
        if (arg0 == null) throw new IllegalArgumentException("url == null");

        com.alibaba.dubbo.common.URL url = arg0;
        String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );

        if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.registry.RegistryFactory) name from url(" + url.toString() + ") use keys([protocol])");

        com.alibaba.dubbo.registry.RegistryFactory extension = (com.alibaba.dubbo.registry.RegistryFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.registry.RegistryFactory.class).getExtension(extName);

        return extension.getRegistry(arg0);
    }
}
```
所以这里registryFactory.getRegistry(registryUrl)用的是ZookeeperRegistryFactory。

先看下getRegistry方法：

```
public Registry getRegistry(URL url) {
    url = url.setPath(RegistryService.class.getName())
            .addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
            .removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
    String key = url.toServiceString();
    // 锁定注册中心获取过程，保证注册中心单一实例
    LOCK.lock();
    try {
        Registry registry = REGISTRIES.get(key);
        if (registry != null) {
            return registry;
        }
        //创建registry，会直接new一个ZookeeperRegistry返回
        registry = createRegistry(url);
        if (registry == null) {
            throw new IllegalStateException("Can not create registry " + url);
        }
        REGISTRIES.put(key, registry);
        return registry;
    } finally {
        // 释放锁
        LOCK.unlock();
    }
}
```
直接前往ZookeeperRegistry中看：

```
public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
    super(url);
    if (url.isAnyHost()) {
        throw new IllegalStateException("registry address == null");
    }
    String group = url.getParameter(Constants.GROUP_KEY, DEFAULT_ROOT);
    if (! group.startsWith(Constants.PATH_SEPARATOR)) {
        group = Constants.PATH_SEPARATOR + group;
    }
    this.root = group;
    zkClient = zookeeperTransporter.connect(url);
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
zookeeperTransport也是生成的代码，默认是zkClient的实现：

```
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

所以zookeeperTransporter.connect(url)就是调用的ZookeeperClient的connect方法。直接返回一个ZkclientZookeeperClient实例。最后添加zkClient的监听。

获取到了Registry实例之后，下一步获取要注册到注册中心的url。然后调用registry.register(registedProviderUrl)注册到注册中心。具体的调用暂先不解析。最后注册一个订阅覆盖配置的监听器，返回Exporter。

# 名词解释
## Invoker
可执行的对象，执行具体的远程调用，能够根据方法名称，参数得到相应的执行结果。

Invocation，包含了需要执行的方法，参数等信息。目前实现类只有RpcInvocation。

有三种类型的Invoker：

- 本地执行类的Invoker。
- 远程通信执行类的Invoker。
- 多个远程通信执行类的Invoker聚合成集群版的Invoker。

以HelloService为例：

- 本地执行类的Invoker：在Server端有HelloServiceImpl实现，要执行该接口，只需要通过反射执行对应的实现类即可。
- 远程通信执行类的Invoker：在Client端要想执行该接口的实现方法，需要先进行远程通信，发送要执行的参数信息给Server端，Server端利用本地执行Invoker的方式执行，最后将结果发送给Client。
- 集群版的Invoker：Client端使用的时候，通过集群版的Invoker操作，Invoker会挑选一个远程通信类型的Invoker来执行。

提供者端的Invoker封装了服务实现类，URL，Type，状态都是只读并且线程安全。通过发起invoke来具体调用服务类。

## ProxyFactory
在服务提供者端，ProxyFactory主要服务的实现统一包装成一个Invoker，Invoker通过反射来执行具体的Service实现对象的方法。默认的实现是JavassistProxyFactory，代码如下：

```
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    // TODO Wrapper类不能正确处理带$的类名
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName, 
                                  Class<?>[] parameterTypes, 
                                  Object[] arguments) throws Throwable {
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```


## Protocol
服务地址的发布和订阅。

Protocol是dubbo中的服务域，只在服务启用时加载，无状态，线程安全，是实体域Invoker暴露和引用的主功能入口，负责Invoker的生命周期管理，是Dubbo中远程服务调用层。

Protocol根据指定协议对外公布服务，当客户端根据协议调用这个服务时，Protocol会将客户端传递过来的Invocation参数交给Invoker去执行。

Protocol加入了远程通信协议，会根据客户端的请求来获取参数Invocation。

```
@Extension("dubbo")
public interface Protocol {

    int getDefaultPort();

    //对于服务提供端，将本地执行类的Invoker通过协议暴漏给外部
    //外部可以通过协议发送执行参数Invocation，然后交给本地Invoker来执行
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    //这个是针对服务消费端的，服务消费者从注册中心获取服务提供者发布的服务信息
    //通过服务信息得知服务提供者使用的协议，然后服务消费者仍然使用该协议构造一个Invoker。这个Invoker是远程通信类的Invoker。
    //执行时，需要将执行信息通过指定协议发送给服务提供者，服务提供者接收到参数Invocation，然后交给服务提供者的本地Invoker来执行
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    void destroy();

}
```

### 关于RegistryProtocol和DubboProtocol的疑惑

以下是官方文档说明：

> 暴露服务:
> 
>  (1) 只暴露服务端口：

> 在没有注册中心，直接暴露提供者的情况下，即：
`<dubbo:service regisrty="N/A" /> or <dubbo:registry address="N/A" />`

> ServiceConfig解析出的URL的格式为：
`dubbo://service-host/com.foo.FooService?version=1.0.0`

> 基于扩展点的Adaptive机制，通过URL的"dubbo://"协议头识别，直接调用DubboProtocol的export()方法，打开服务端口。
>
>  (2) 向注册中心暴露服务：

> 在有注册中心，需要注册提供者地址的情况下，即：
`<dubbo:registry address="zookeeper://10.20.153.10:2181" />`

> ServiceConfig解析出的URL的格式为：
`registry://registry-host/com.alibaba.dubbo.registry.RegistryService?export=URL.encode("dubbo://service-host/com.foo.FooService?version=1.0.0")`

> 基于扩展点的Adaptive机制，通过URL的"registry://"协议头识别，就会调用RegistryProtocol的export()方法，将export参数中的提供者URL，先注册到注册中心，再重新传给Protocol扩展点进行暴露：
`dubbo://service-host/com.foo.FooService?version=1.0.0`

> 基于扩展点的Adaptive机制，通过提供者URL的"dubbo://"协议头识别，就会调用DubboProtocol的export()方法，打开服务端口。

RegistryProtocol，注册中心协议集成，装饰真正暴露引用服务的协议，增强注册发布功能。

ServiceConfig中的protocol是被多层装饰的Protocol，是DubboProtocol+RegistryProtocol+ProtocolListenerWrapper+ProtocolFilterWrapper。

- ProtocolFilterWrapper负责初始化invoker所有的Filter。
- ProtocolListenerWrapper负责初始化暴露或引用服务的监听器。
- RegistryProtocol负责注册服务到注册中心和向注册中心订阅服务。
- DubboProtocol负责服务的具体暴露与引用，也负责网络传输层，信息交换层的初始化，以及底层NIO框架的初始化。


## Exporter
负责invoker的生命周期，包含一个Invoker对象，可以撤销服务。

## Exchanger
负责数据交换和网络通信的组件。每个Invoker都维护了一个ExchangeClient的 引用，并通过它和远端server进行通信。