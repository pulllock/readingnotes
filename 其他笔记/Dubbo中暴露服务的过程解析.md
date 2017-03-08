dubbo暴露服务有两种情况，一种是设置了延迟暴露（比如delay="5000"），另外一种是没有设置延迟暴露或者延迟设置为-1（delay="-1"）：

- 设置了延迟暴露，dubbo在Spring实例化bean（initializeBean）的时候会对实现了InitializingBean的类进行回调，回调方法是afterPropertySet()，如果设置了延迟暴露，dubbo在这个方法中进行服务的发布。
- 没有设置延迟或者延迟为-1，dubbo会在Spring实例化完bean之后，在刷新容器最后一步发布ContextRefreshEvent事件的时候，通知实现了ApplicationListener的类进行回调onApplicationEvent，dubbo会在这个方法中发布服务。

但是不管延迟与否，都是使用ServiceConfig的export()方法进行服务的暴露。使用export初始化的时候会将Bean对象转换成URL格式，所有Bean属性转换成URL的参数。

## 过程
以没有设置延迟暴露熟属性的过程为例。

- 当Spring容器实例化bean完成，走到最后一步发布ContextRefreshEvent事件的时候，ServiceBean会执行onApplicationEvent方法，该方法调用ServiceConfig的export方法。

- ServiceConfig初始化的时候，会先初始化静态变量protocol和proxyFactory，这两个变量初始化的结果是通过dubbo的spi扩展机制得到的。

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

- export方法先判断是否需要延迟暴露（这里我们使用的是不延迟暴露），然后执行doExport方法。
- doExport方法先执行一系列的检查方法，然后调用doExportUrls方法。检查方法会检测dubbo的配置是否在Spring配置文件中声明，没有的话读取properties文件初始化。
- doExportUrls方法先调用loadRegistries获取所有的注册中心url，然后遍历调用doExportUrlsFor1Protocol方法。对于在标签中指定了registry属性的Bean，会在加载BeanDefinition的时候就加载了注册中心。
	
    获取注册中心url，会把注册的信息都放在一个URL对象中，一个URL内容如下：
    
    ```
    registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=dubbo-provider&application.version=1.0&dubbo=2.5.3&environment=product&organization=china&owner=cheng.xi&pid=2939&registry=zookeeper&timestamp=1488898049284
    ```
    
- doExportUrlsFor1Protocol根据不同的协议将服务以URL形式暴露。如果scope配置为none则不暴露，如果服务未配置成remote，则本地暴露exportLocal，如果未配置成local，则注册服务registryProcotol。
	
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
     //暴露服务invoker
     //调用Protocol生成的适配类的export方法
     //这里的protocol是上面列出的生成的代码
     Exporter<?> exporter = protocol.export(invoker);
    ```
    **关于Invoker，Exporter等的解释参见下面的内容。**
    
- 接着就是protocol的发布
	
   ` Exporter<?> exporter = protocol.export(invoker);`
   
   这里还是以dubbo协议为例子，对应的Protocol为DubboProtocol，使用SPI机制获取到DubboProtocol的过程中，就已经进行了ProtocolFilterWrapper、ProtocolListenerWrapper、RegistryProtocol等的包装。
  	
    在invoker中包含了registryUrl，这个url中的Protocol值为registry，所以会调用RegistryProtocol去暴露服务。发布过程首先会经过RegistryProtocol：
   
   ```
   public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    //这里利用具体的Protocol进行服务的导出，我们这里使用的是DubboProtocol
    //暴露服务，启动提供者
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
    //获取注册服务Registry，将url注册到注册中心上，供消费者订阅。
    final Registry registry = getRegistry(originInvoker);
    //获取到服务提供者的地址，注册到注册中心
    final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
    registry.register(registedProviderUrl);
    // 订阅override数据
    // FIXME 提供者订阅时，会影响同一JVM即暴露服务，又引用同一服务的的场景，因为subscribed以服务名为缓存的key，导致订阅信息覆盖。
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    //服务提供者订阅，对这个服务提供者的地址进行监听。
    //当节点发生变化时，注册中心会通知各个节点，实际调用OverrideListener.notify()方法重新暴露服务。
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    //保证每次export都返回一个新的exporter实例
    //包装export并重写unexport方法，可以取消订阅
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

DubboProtocol服务的导出过程：

```
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
	//获取url
    URL url = invoker.getUrl();
    // export service.
    String key = serviceKey(url);
    //创建一个DubboExporter来封装invoker
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
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

    openServer(url);

    return exporter;
}
```

```
private void openServer(URL url) {
    // find server.
    String key = url.getAddress();
    //client 也可以暴露一个只有server可以调用的服务。
    boolean isServer = url.getParameter(Constants.IS_SERVER_KEY,true);
    if (isServer) {
        ExchangeServer server = serverMap.get(key);
        if (server == null) {
        	//serverMap中，key是host+port，value是一个ExchangeServer
            //createServer打开服务，默认使用netty
            serverMap.put(key, createServer(url));
        } else {
            //server支持reset,配合override功能使用
            server.reset(url);
        }
    }
}
```

createServer打开服务，传入requestHandler。

server = Exchangers.bind(url,requestHandler);

返回exported对象。

ExchangeHandler.reply()方法处理消费者的消息，调用invoker.invoke执行具体的方法。

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



## Exporter
负责invoker的生命周期，包含一个Invoker对象，可以撤销服务。

## Exchanger
负责数据交换和网络通信的组件。每个Invoker都维护了一个ExchangeClient的 引用，并通过它和远端server进行通信。