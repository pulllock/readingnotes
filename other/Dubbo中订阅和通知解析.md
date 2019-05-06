Dubbo中关于服务的订阅和通知主要发生在服务提供方暴露服务的过程和服务消费方初始化时候引用服务的过程中。

# 服务引用过程中的订阅和通知
在服务消费者初始化的过程中，会有一步是进行服务的引用，具体的代码是在RegistryProtocol的refer方法：

```
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
    //在这一步获取注册中心实例的过程中，也会有notify的操作。（这里省略）
    Registry registry = registryFactory.getRegistry(url);
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }

    // group="a,b" or group="*"
    Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
    String group = qs.get(Constants.GROUP_KEY);
    if (group != null && group.length() > 0 ) {
        if ( ( Constants.COMMA_SPLIT_PATTERN.split( group ) ).length > 1
                || "*".equals( group ) ) {
            return doRefer( getMergeableCluster(), registry, type, url );
        }
    }
    return doRefer(cluster, registry, type, url);
}
```
在refer方法中有一步是获取注册中心实例，这一步中也会有一个notify操作，先暂时不解释。接着就是doRefer方法：

```
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    directory.setRegistry(registry);
    directory.setProtocol(protocol);
    //订阅的url
    URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, NetUtils.getLocalHost(), 0, type.getName(), directory.getUrl().getParameters());
    if (! Constants.ANY_VALUE.equals(url.getServiceInterface())
            && url.getParameter(Constants.REGISTER_KEY, true)) {
        //服务消费方向注册中心注册自己，供其他层使用，比如服务治理
        registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                Constants.CHECK_KEY, String.valueOf(false)));
    }
    //订阅服务提供方
    //同时订阅了三种类型providers，routers，configurators。
    directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY, 
            Constants.PROVIDERS_CATEGORY 
            + "," + Constants.CONFIGURATORS_CATEGORY 
            + "," + Constants.ROUTERS_CATEGORY));
    return cluster.join(directory);
}
```
在doRefer方法中服务消费者会订阅服务，同时订阅了三种类型：providers，routers，configurators。

接续看directory.subscribe订阅方法，这里directory是RegistryDirectory：

```
public void subscribe(URL url) {
	//设置消费者url
    setConsumerUrl(url);
    //订阅
    //url为订阅条件，不能为空
    //第二个参数this，是变更事件监听器，不允许为空，RegistryDirectory实现了NotifyListener接口，因此是一个事件监听器
    registry.subscribe(url, this);
}
```
这里registry是ZookeeperRegistry，在ZookeeperRegistry调用subscribe处理之前会先经过AbstractRegistry的处理，然后经过FailbackRegistry处理，在FailbackRegistry中会调用ZookeeperRegistry的doSubscribe方法。

首先看下AbstractRegistry中subscribe方法：

```
public void subscribe(URL url, NotifyListener listener) {
    if (url == null) {
        throw new IllegalArgumentException("subscribe url == null");
    }
    if (listener == null) {
        throw new IllegalArgumentException("subscribe listener == null");
    }
    //从缓存中获取已经订阅的url的监听器
    Set<NotifyListener> listeners = subscribed.get(url);
    if (listeners == null) {
        subscribed.putIfAbsent(url, new ConcurrentHashSet<NotifyListener>());
        listeners = subscribed.get(url);
    }
    //将当前监听器添加到监听器的set中
    listeners.add(listener);
}
```

然后是FailbackRegistry的subscribe方法：

```
public void subscribe(URL url, NotifyListener listener) {
	//上面AbstractRegistry的处理
    super.subscribe(url, listener);
    //移除订阅失败的
    removeFailedSubscribed(url, listener);
    try {
        // 向服务器端发送订阅请求
        //子类实现，我们这里使用的是ZookeeperRegistry
        doSubscribe(url, listener);
    } catch (Exception e) {
        Throwable t = e;

        List<URL> urls = getCacheUrls(url);
        if (urls != null && urls.size() > 0) {
        	//订阅失败，进行通知，重试
            notify(url, listener, urls);
        } else {
            // 如果开启了启动时检测，则直接抛出异常
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true);
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                if(skipFailback) {
                    t = t.getCause();
                }
                throw new IllegalStateException("Failed to subscribe " + url + ", cause: " + t.getMessage(), t);
            }
        }

        // 将失败的订阅请求记录到失败列表，定时重试
        addFailedSubscribed(url, listener);
    }
}
```
这里总共进行了一下几件事情：

- AbstractRegistry的处理
- 移除订阅失败的
- 由具体的子类向服务器端发送订阅请求
- 如果订阅发生失败了，尝试获取缓存url，然后进行失败通知或者如果开启了启动时检测，则直接抛出异常
- 将失败的订阅请求记录到失败列表，定时重试

主要看下子类向服务器段发送订阅请求的步骤，在ZookeeperRegistry的doSubscribe方法中：

```
protected void doSubscribe(final URL url, final NotifyListener listener) {
    try {
        if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {//这里暂时没用到先不解释
            String root = toRootPath();
            ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
            if (listeners == null) {
                zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                listeners = zkListeners.get(url);
            }
            ChildListener zkListener = listeners.get(listener);
            if (zkListener == null) {
                listeners.putIfAbsent(listener, new ChildListener() {
                    public void childChanged(String parentPath, List<String> currentChilds) {
                        for (String child : currentChilds) {
                            child = URL.decode(child);
                            if (! anyServices.contains(child)) {
                                anyServices.add(child);
                                subscribe(url.setPath(child).addParameters(Constants.INTERFACE_KEY, child, 
                                        Constants.CHECK_KEY, String.valueOf(false)), listener);
                            }
                        }
                    }
                });
                zkListener = listeners.get(listener);
            }
            zkClient.create(root, false);
            List<String> services = zkClient.addChildListener(root, zkListener);
            if (services != null && services.size() > 0) {
                for (String service : services) {
                    service = URL.decode(service);
                    anyServices.add(service);
                    subscribe(url.setPath(service).addParameters(Constants.INTERFACE_KEY, service, 
                            Constants.CHECK_KEY, String.valueOf(false)), listener);
                }
            }
        } else {
            List<URL> urls = new ArrayList<URL>();
            //这里的path分别为providers，routers，configurators三种
            for (String path : toCategoriesPath(url)) {
            	//根据url获取对应的监听器map
                ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                if (listeners == null) {
                    zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                    listeners = zkListeners.get(url);
                }
                //根据我们的listener获取一个ChildListener实例
                ChildListener zkListener = listeners.get(listener);
                //没有的话就创建一个ChildListener实例。
                if (zkListener == null) {
                    listeners.putIfAbsent(listener, new ChildListener() {
                        public void childChanged(String parentPath, List<String> currentChilds) {
                            ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
                        }
                    });
                    zkListener = listeners.get(listener);
                }
                //根据path在Zookeeper中创建节点，这里就是订阅服务
                zkClient.create(path, false);
                //这里zkClient是dubbo的ZookeeperClient，在addChildListener中会转化为ZkClient的Listener
                List<String> children = zkClient.addChildListener(path, zkListener);
                if (children != null) {
                    urls.addAll(toUrlsWithEmpty(url, path, children));
                }
            }
            //订阅完成之后，进行通知
            notify(url, listener, urls);
        }
    } catch (Throwable e) {
        throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```
上面主要是分别对providers，routers，configurators三种不同类型的进行订阅，也就是往zookeeper中注册节点，注册之前先给url添加监听器。最后是订阅完之后进行通知。

notify方法，这里notify方法实现是在ZookeeperRegistry的父类FailbackRegistry中：

```
protected void notify(URL url, NotifyListener listener, List<URL> urls) {
    if (url == null) {
        throw new IllegalArgumentException("notify url == null");
    }
    if (listener == null) {
        throw new IllegalArgumentException("notify listener == null");
    }
    try {
    	//doNotify方法中没做处理，直接调用父类的notify方法
        doNotify(url, listener, urls);
    } catch (Exception t) {
        // 将失败的通知请求记录到失败列表，定时重试
        Map<NotifyListener, List<URL>> listeners = failedNotified.get(url);
        if (listeners == null) {
            failedNotified.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, List<URL>>());
            listeners = failedNotified.get(url);
        }
        listeners.put(listener, urls);
    }
}
```

看下AbstractRegistry的notify方法：

```
protected void notify(URL url, NotifyListener listener, List<URL> urls) {
    Map<String, List<URL>> result = new HashMap<String, List<URL>>();
    //获取catagory列表，providers，routers，configurators
    for (URL u : urls) {
        if (UrlUtils.isMatch(url, u)) {
            String category = u.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
            List<URL> categoryList = result.get(category);
            if (categoryList == null) {
                categoryList = new ArrayList<URL>();
                result.put(category, categoryList);
            }
            categoryList.add(u);
        }
    }
    if (result.size() == 0) {
        return;
    }
    //已经通知过
    Map<String, List<URL>> categoryNotified = notified.get(url);
    if (categoryNotified == null) {
        notified.putIfAbsent(url, new ConcurrentHashMap<String, List<URL>>());
        categoryNotified = notified.get(url);
    }
    for (Map.Entry<String, List<URL>> entry : result.entrySet()) {
    	//providers，routers，configurators中的一个
        String category = entry.getKey();
        List<URL> categoryList = entry.getValue();
        categoryNotified.put(category, categoryList);
        saveProperties(url);
        //还记得刚开始的时候，listener参数么，这里listener是RegistryDirectory
        listener.notify(categoryList);
    }
}
```

继续看RegistryDirectory的notify方法：

```
public synchronized void notify(List<URL> urls) {
	//三种类型分开
    List<URL> invokerUrls = new ArrayList<URL>();
    List<URL> routerUrls = new ArrayList<URL>();
    List<URL> configuratorUrls = new ArrayList<URL>();
    for (URL url : urls) {
        String protocol = url.getProtocol();
        String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
        if (Constants.ROUTERS_CATEGORY.equals(category) 
                || Constants.ROUTE_PROTOCOL.equals(protocol)) {
            routerUrls.add(url);
        } else if (Constants.CONFIGURATORS_CATEGORY.equals(category) 
                || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
            configuratorUrls.add(url);
        } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
            invokerUrls.add(url);
        } else {
        }
    }
    // configurators
    //更新缓存的服务提供方配置规则
    if (configuratorUrls != null && configuratorUrls.size() >0 ){
        this.configurators = toConfigurators(configuratorUrls);
    }
    // routers
    //更新缓存的路由配置规则
    if (routerUrls != null && routerUrls.size() >0 ){
        List<Router> routers = toRouters(routerUrls);
        if(routers != null){ // null - do nothing
            setRouters(routers);
        }
    }
    List<Configurator> localConfigurators = this.configurators; // local reference
    // 合并override参数
    this.overrideDirectoryUrl = directoryUrl;
    if (localConfigurators != null && localConfigurators.size() > 0) {
        for (Configurator configurator : localConfigurators) {
            this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
        }
    }
    // providers
    //重建invoker实例
    refreshInvoker(invokerUrls);
}
```

最重要的重建invoker实例，在服务引用的文章中已经介绍过，不再重复，还有上面说省略的`获取注册中心实例的过程中，也会有notify的操作。（这里省略）`这里也是进行了invoker实例的重建。


# 暴露服务过程中的订阅和通知
服务暴露过程中的订阅在RegistryProtocol的export方法中：

```
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    //export invoker
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
    //registry provider
    final Registry registry = getRegistry(originInvoker);
    final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
    registry.register(registedProviderUrl);
    // 订阅override数据
    // FIXME 提供者订阅时，会影响同一JVM即暴露服务，又引用同一服务的的场景，因为subscribed以服务名为缓存的key，导致订阅信息覆盖。
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
    //OverrideListener是RegistryProtocol的内部类
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    //订阅override数据
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    //保证每次export都返回一个新的exporter实例
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
registry.subscribe订阅override数据，会首先经过AbstractRegistry处理，然后经过FailbackRegistry处理。处理方法在上面消费者发布订阅的讲解中都已经介绍。往下的步骤基本相同，不同之处在于AbstractRegistry的notify方法：

```
protected void notify(URL url, NotifyListener listener, List<URL> urls) {
    Map<String, List<URL>> result = new HashMap<String, List<URL>>();
    //获取catagory列表，providers，routers，configurators
    for (URL u : urls) {
        if (UrlUtils.isMatch(url, u)) {
            String category = u.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
            List<URL> categoryList = result.get(category);
            if (categoryList == null) {
                categoryList = new ArrayList<URL>();
                result.put(category, categoryList);
            }
            categoryList.add(u);
        }
    }
    if (result.size() == 0) {
        return;
    }
    //已经通知过
    Map<String, List<URL>> categoryNotified = notified.get(url);
    if (categoryNotified == null) {
        notified.putIfAbsent(url, new ConcurrentHashMap<String, List<URL>>());
        categoryNotified = notified.get(url);
    }
    for (Map.Entry<String, List<URL>> entry : result.entrySet()) {
    	//providers，routers，configurators中的一个
        String category = entry.getKey();
        List<URL> categoryList = entry.getValue();
        categoryNotified.put(category, categoryList);
        saveProperties(url);
        //对于消费者来说这里listener是RegistryDirectory
        //而对于服务提供者来说这里是OverrideListener，是RegistryProtocol的内部类
        listener.notify(categoryList);
    }
}
```

接下来看OverrideListener的notify方法：

```
/*
 *  provider 端可识别的override url只有这两种.
 *  override://0.0.0.0/serviceName?timeout=10
 *  override://0.0.0.0/?timeout=10
 */
public void notify(List<URL> urls) {
    List<URL> result = null;
    for (URL url : urls) {
        URL overrideUrl = url;
        if (url.getParameter(Constants.CATEGORY_KEY) == null
                && Constants.OVERRIDE_PROTOCOL.equals(url.getProtocol())) {
            // 兼容旧版本
            overrideUrl = url.addParameter(Constants.CATEGORY_KEY, Constants.CONFIGURATORS_CATEGORY);
        }
        if (! UrlUtils.isMatch(subscribeUrl, overrideUrl)) {
            if (result == null) {
                result = new ArrayList<URL>(urls);
            }
            result.remove(url);
            logger.warn("Subsribe category=configurator, but notifed non-configurator urls. may be registry bug. unexcepted url: " + url);
        }
    }
    if (result != null) {
        urls = result;
    }
    this.configurators = RegistryDirectory.toConfigurators(urls);
    List<ExporterChangeableWrapper<?>> exporters = new ArrayList<ExporterChangeableWrapper<?>>(bounds.values());
    for (ExporterChangeableWrapper<?> exporter : exporters){
        Invoker<?> invoker = exporter.getOriginInvoker();
        final Invoker<?> originInvoker ;
        if (invoker instanceof InvokerDelegete){
            originInvoker = ((InvokerDelegete<?>)invoker).getInvoker();
        }else {
            originInvoker = invoker;
        }

        URL originUrl = RegistryProtocol.this.getProviderUrl(originInvoker);
        URL newUrl = getNewInvokerUrl(originUrl, urls);

        if (! originUrl.equals(newUrl)){
        	//对修改了url的invoker重新export
            RegistryProtocol.this.doChangeLocalExport(originInvoker, newUrl);
        }
    }
}
```
这里也是对Invoker重新进行了引用。