> Directory代表多个Invoker，可以把它看成List<Invoker>，但与List不同的是，它的值可能是动态变化的，比如注册中心推送变更。Cluster将Directory中的多个Invoker伪装成一个Invoker，对上层透明，伪装过程包含了容错逻辑，调用失败后，重试另一个。

上面是文档上对Directory的解释。

# Directory接口

Directory接口继承了Node接口：

```
public interface Directory<T> extends Node {
    //获取服务类型
    Class<T> getInterface();

    //invoker列表，服务的列表
    List<Invoker<T>> list(Invocation invocation) throws RpcException;
}
```

# AbstractDirectory

默认实现为AbstractDirectory：

```
public abstract class AbstractDirectory<T> implements Directory<T> {

    // 日志输出
    private static final Logger logger = LoggerFactory.getLogger(AbstractDirectory.class);
	//服务url
    private final URL url ;
    private volatile boolean destroyed = false;
	//消费者url
    private volatile URL consumerUrl ;
    //路由
	private volatile List<Router> routers;
    
    public AbstractDirectory(URL url) {
        this(url, null);
    }
    
    public AbstractDirectory(URL url, List<Router> routers) {
    	this(url, url, routers);
    }
    
    public AbstractDirectory(URL url, URL consumerUrl, List<Router> routers) {
        if (url == null)
            throw new IllegalArgumentException("url == null");
        this.url = url;
        this.consumerUrl = consumerUrl;
        setRouters(routers);
    }
    //对list方法的默认实现
    public List<Invoker<T>> list(Invocation invocation) throws RpcException {
        if (destroyed){
            throw new RpcException("Directory already destroyed .url: "+ getUrl());
        }
        //获取Invoker列表的具体实现由具体子类实现
        List<Invoker<T>> invokers = doList(invocation);
        //路由
        List<Router> localRouters = this.routers; // local reference
        if (localRouters != null && localRouters.size() > 0) {
            for (Router router: localRouters){
                try {
                    if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, true)) {
                    	//路由
                        invokers = router.route(invokers, getConsumerUrl(), invocation);
                    }
                } catch (Throwable t) {
                    logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
                }
            }
        }
        return invokers;
    }
    
    public URL getUrl() {
        return url;
    }
    
    public List<Router> getRouters(){
        return routers;
    }

	public URL getConsumerUrl() {
		return consumerUrl;
	}

	public void setConsumerUrl(URL consumerUrl) {
		this.consumerUrl = consumerUrl;
	}
	//构造中调用的设置路由的方法
    protected void setRouters(List<Router> routers){
        // copy list
        routers = routers == null ? new  ArrayList<Router>() : new ArrayList<Router>(routers);
        // append url router
    	String routerkey = url.getParameter(Constants.ROUTER_KEY);
        //指定了router，就使用制定的router来获取扩展实现
        if (routerkey != null && routerkey.length() > 0) {
            RouterFactory routerFactory = ExtensionLoader.getExtensionLoader(RouterFactory.class).getExtension(routerkey);
            routers.add(routerFactory.getRouter(url));
        }
        // append mock invoker selector
        routers.add(new MockInvokersSelector());
        Collections.sort(routers);
    	this.routers = routers;
    }

    public boolean isDestroyed() {
        return destroyed;
    }

    public void destroy(){
        destroyed = true;
    }
	//子类实现具体的获取invoker列表
    protected abstract List<Invoker<T>> doList(Invocation invocation) throws RpcException ;

}
```

Directory具体的实现有两个RegistryDirectory注册目录服务和StaticDirectory静态目录服务。

# RegistryDirectory

RegistryDirectory实现了NotifyListener接口，因此他本身也是一个监听器，可以在服务变更时接受通知，消费方要调用远程服务，会向注册中心订阅这个服务的所有的服务提供方，订阅的时候会调用notify方法，进行invoker实例的重新生成，也就是服务的重新引用。在服务提供方有变动时，也会调用notify方法，有关notify方法在Dubbo中订阅和通知解析那篇文章中已经解释，不做重复。subscribe方法也不做重复解释。

# StaticDirectory
静态目录服务。