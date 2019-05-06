Dubbo中的Cluster可以将多个服务提供方伪装成一个提供方，具体也就是将Directory中的多个Invoker伪装成一个Invoker，在伪装的过程中包含了容错的处理，负载均衡的处理和路由的处理。这篇文章介绍下集群相关的东西，开始先对着文档解释下容错模式，负载均衡，路由等概念，然后解析下源码的处理。（稍微有点乱，心情不太好，不适合分析源码。）

# 集群的容错模式
## Failover Cluster
这是dubbo中默认的集群容错模式

- 失败自动切换，当出现失败，重试其它服务器。
- 通常用于读操作，但重试会带来更长延迟。
- 可通过retries="2"来设置重试次数(不含第一次)。

## Failfast Cluster

- 快速失败，只发起一次调用，失败立即报错。
- 通常用于非幂等性的写操作，比如新增记录。

## Failsafe Cluster

- 失败安全，出现异常时，直接忽略。
- 通常用于写入审计日志等操作。

## Failback Cluster

- - 失败自动恢复，后台记录失败请求，定时重发。
- 通常用于消息通知操作。

## Forking Cluster

- 并行调用多个服务器，只要一个成功即返回。
- 通常用于实时性要求较高的读操作，但需要浪费更多服务资源。
- 可通过forks="2"来设置最大并行数。

## Broadcast Cluster
- 广播调用所有提供者，逐个调用，任意一台报错则报错。(2.1.0开始支持)
- 通常用于通知所有提供者更新缓存或日志等本地资源信息。

# 负载均衡
dubbo默认的负载均衡策略是random，随机调用。

## Random LoadBalance

- 随机，按权重设置随机概率。
- 在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重。

## RoundRobin LoadBalance

- 轮循，按公约后的权重设置轮循比率。
- 存在慢的提供者累积请求问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上。

## LeastActive LoadBalance

- 最少活跃调用数，相同活跃数的随机，活跃数指调用前后计数差。
- 使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大。

## ConsistentHash LoadBalance

- 一致性Hash，相同参数的请求总是发到同一提供者。
- 当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。
- 缺省只对第一个参数Hash。
- 缺省用160份虚拟节点。

# 集群相关源码解析
回想一下在服务消费者初始化的过程中，在引用远程服务的那一步，也就是RegistryProtocol的refer方法中，调用了doRefer方法，doRefer方法中第一个参数就是cluster，我们就从这里开始解析。RegistryProtocol的refer方法：

```
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
    //根据url获取注册中心实例
    //这一步连接注册中心，并把消费者注册到注册中心
    Registry registry = registryFactory.getRegistry(url);
    //对注册中心服务的处理
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }
	//以下是我们自己定义的业务的服务处理
    // group="a,b" or group="*"
    Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
    String group = qs.get(Constants.GROUP_KEY);
    //服务需要合并不同实现
    if (group != null && group.length() > 0 ) {
        if ( ( Constants.COMMA_SPLIT_PATTERN.split( group ) ).length > 1
                || "*".equals( group ) ) {
            return doRefer( getMergeableCluster(), registry, type, url );
        }
    }
    //这里参数cluster是集群的适配类，代码在下面
    return doRefer(cluster, registry, type, url);
}
```

接着看doRefer，真正去做服务引用的方法：

```
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
	//Directory中是Invoker的集合，相当于一个List
    //也就是说这里面存放了多个Invoker，那么我们该调用哪一个呢？
    //该调用哪一个Invoker的工作就是Cluster来处理的
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    directory.setRegistry(registry);
    directory.setProtocol(protocol);
    URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, NetUtils.getLocalHost(), 0, type.getName(), directory.getUrl().getParameters());
    if (! Constants.ANY_VALUE.equals(url.getServiceInterface())
            && url.getParameter(Constants.REGISTER_KEY, true)) {
        
 //到注册中心注册服务       registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                Constants.CHECK_KEY, String.valueOf(false)));
    }
    
 //订阅服务，注册中心会推送服务消息给消费者，消费者会再次进行服务的引用。   directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY, 
            Constants.PROVIDERS_CATEGORY 
            + "," + Constants.CONFIGURATORS_CATEGORY 
            + "," + Constants.ROUTERS_CATEGORY));
    //服务的引用和变更全部由Directory异步完成
    //Directory中可能存在多个Invoker
    //而Cluster会把多个Invoker伪装成一个Invoker
    //这一步就是做这个事情的
    return cluster.join(directory);
}
```

## 集群处理的入口
入口就是在doRefer的时候最后一步：`cluster.join(directory);`。

首先解释下cluster，这个是根据dubbo的扩展机制生成的，在RegistryProtocol中有一个setCluster方法，根据扩展机制可以知道，这是注入Cluster的地方，代码如下：

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
可以看到，如果我们没有配置集群策略的话，默认是用failover模式，在Cluster接口的注解上`@SPI(FailoverCluster.NAME)`也可以看到默认是failover。

继续执行cluster.join方法，会首先进入MockClusterWrapper的join方法：

```
public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
	//先执行FailoverCluster的join方法处理
    //然后将Directory和返回的Invoker封装成一个MockCluster
    return new MockClusterInvoker<T>(directory,
            this.cluster.join(directory));
}
```

看下Failover的join方法：

```
public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
	//直接返回一个FailoverClusterInvoker的实例
    return new FailoverClusterInvoker<T>(directory);
}
```

到这里就算把Invoker都封装好了，返回的Invoker是一个MockClusterInvoker，MockClusterInvoker内部包含一个Directory和一个FailoverClusterInvoker。

Invoker都封装好了之后，就是创建代理，然后使用代理调用我们的要调用的方法。

## 调用方法时集群的处理
在进行具体方法调用的时候，代理中会`invoker.invoke()`，这里Invoker就是我们上面封装好的MockClusterInvoker，所以首先进入MockClusterInvoker的invoke方法：

```
public Result invoke(Invocation invocation) throws RpcException {
    Result result = null;
	//我们没配置mock，所以这里为false
    //Mock通常用于服务降级
    String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), Constants.MOCK_KEY, Boolean.FALSE.toString()).trim(); 
    //没有使用mock
    if (value.length() == 0 || value.equalsIgnoreCase("false")){
        //这里的invoker是FailoverClusterInvoker
        result = this.invoker.invoke(invocation);
    } else if (value.startsWith("force")) {
    	//mock=force:return+null
        //表示消费方对方法的调用都直接返回null，不发起远程调用
        //可用于屏蔽不重要服务不可用的时候，对调用方的影响
        //force:direct mock
        result = doMockInvoke(invocation, null);
    } else {
    	//mock=fail:return+null
        //表示消费方对该服务的方法调用失败后，再返回null，不抛异常
        //可用于对不重要服务不稳定的时候，忽略对调用方的影响
        //fail-mock
        try {
            result = this.invoker.invoke(invocation);
        }catch (RpcException e) {
            if (e.isBiz()) {
                throw e;
            } else {
                result = doMockInvoke(invocation, e);
            }
        }
    }
    return result;
}
```
我们这里么有配置mock属性。首先进入的是AbstractClusterInvoker的incoke方法：

```
public Result invoke(final Invocation invocation) throws RpcException {
	//检查是否已经被销毁
    checkWheatherDestoried();
	//可以看到这里该处理负载均衡的问题了
    LoadBalance loadbalance;
	//根据invocation中的信息从Directory中获取Invoker列表
    //这一步中会进行路由的处理
    List<Invoker<T>> invokers = list(invocation);
    if (invokers != null && invokers.size() > 0) {
    	//使用扩展机制，加载LoadBalance的实现类，默认使用的是random
        //我们这里得到的就是RandomLoadBalance
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                .getMethodParameter(invocation.getMethodName(),Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
    } else {
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(Constants.DEFAULT_LOADBALANCE);
    }
    //异步操作默认添加invocation id
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    //调用具体的实现类的doInvoke方法，这里是FailoverClusterInvoker
    return doInvoke(invocation, invokers, loadbalance);
}
```

看下FailoverClusterInvoker的invoke方法：

```
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
	//Invoker列表
    List<Invoker<T>> copyinvokers = invokers;
    //确认下Invoker列表不为空
    checkInvokers(copyinvokers, invocation);
    //重试次数
    int len = getUrl().getMethodParameter(invocation.getMethodName(), Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
    if (len <= 0) {
        len = 1;
    }
    // retry loop.
    RpcException le = null; // last exception.
    List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers.
    Set<String> providers = new HashSet<String>(len);
    for (int i = 0; i < len; i++) {
        //重试时，进行重新选择，避免重试时invoker列表已发生变化.
        //注意：如果列表发生了变化，那么invoked判断会失效，因为invoker示例已经改变
        if (i > 0) {
            checkWheatherDestoried();
            copyinvokers = list(invocation);
            //重新检查一下
            checkInvokers(copyinvokers, invocation);
        }
        //使用loadBalance选择一个Invoker返回
        Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
        invoked.add(invoker);
        RpcContext.getContext().setInvokers((List)invoked);
        try {
        	//使用选择的结果Invoker进行调用，返回结果
            Result result = invoker.invoke(invocation);
            return result;
        } catch (RpcException e) {。。。} finally {
            providers.add(invoker.getUrl().getAddress());
        }
    }
    throw new RpcException(。。。);
}
```

先看下使用loadbalance选择invoker的select方法：

```
protected Invoker<T> select(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
    if (invokers == null || invokers.size() == 0)
        return null;
    String methodName = invocation == null ? "" : invocation.getMethodName();

    //sticky，滞连接用于有状态服务，尽可能让客户端总是向同一提供者发起调用，除非该提供者挂了，再连另一台。
    boolean sticky = invokers.get(0).getUrl().getMethodParameter(methodName,Constants.CLUSTER_STICKY_KEY, Constants.DEFAULT_CLUSTER_STICKY) ;
    {
        //ignore overloaded method
        if ( stickyInvoker != null && !invokers.contains(stickyInvoker) ){
            stickyInvoker = null;
        }
        //ignore cucurrent problem
        if (sticky && stickyInvoker != null && (selected == null || !selected.contains(stickyInvoker))){
            if (availablecheck && stickyInvoker.isAvailable()){
                return stickyInvoker;
            }
        }
    }
    Invoker<T> invoker = doselect(loadbalance, invocation, invokers, selected);

    if (sticky){
        stickyInvoker = invoker;
    }
    return invoker;
}
```

doselect方法：

```
private Invoker<T> doselect(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
    if (invokers == null || invokers.size() == 0)
        return null;
    //只有一个invoker，直接返回，不需要处理
    if (invokers.size() == 1)
        return invokers.get(0);
    // 如果只有两个invoker，退化成轮循
    if (invokers.size() == 2 && selected != null && selected.size() > 0) {
        return selected.get(0) == invokers.get(0) ? invokers.get(1) : invokers.get(0);
    }
    //使用loadBalance进行选择
    Invoker<T> invoker = loadbalance.select(invokers, getUrl(), invocation);

    //如果 selected中包含（优先判断） 或者 不可用&&availablecheck=true 则重试.
    if( (selected != null && selected.contains(invoker))
            ||(!invoker.isAvailable() && getUrl()!=null && availablecheck)){
        try{
        	//重新选择
            Invoker<T> rinvoker = reselect(loadbalance, invocation, invokers, selected, availablecheck);
            if(rinvoker != null){
                invoker =  rinvoker;
            }else{
                //看下第一次选的位置，如果不是最后，选+1位置.
                int index = invokers.indexOf(invoker);
                try{
                    //最后在避免碰撞
                    invoker = index <invokers.size()-1?invokers.get(index+1) :invoker;
                }catch (Exception e) {。。。 }
            }
        }catch (Throwable t){。。。}
    }
    return invoker;
} 
```

接着看使用loadBalance进行选择，首先进入AbstractLoadBalance的select方法：

```
 public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    if (invokers == null || invokers.size() == 0)
        return null;
    if (invokers.size() == 1)
        return invokers.get(0);
    //	进行选择，具体的子类实现，我们这里是RandomLoadBalance
    return doSelect(invokers, url, invocation);
}
```

接着去RandomLoadBalance中查看：

```
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    int length = invokers.size(); // 总个数
    int totalWeight = 0; // 总权重
    boolean sameWeight = true; // 权重是否都一样
    for (int i = 0; i < length; i++) {
        int weight = getWeight(invokers.get(i), invocation);
        totalWeight += weight; // 累计总权重
        if (sameWeight && i > 0
                && weight != getWeight(invokers.get(i - 1), invocation)) {
            sameWeight = false; // 计算所有权重是否一样
        }
    }
    if (totalWeight > 0 && ! sameWeight) {
        // 如果权重不相同且权重大于0则按总权重数随机
        int offset = random.nextInt(totalWeight);
        // 并确定随机值落在哪个片断上
        for (int i = 0; i < length; i++) {
            offset -= getWeight(invokers.get(i), invocation);
            if (offset < 0) {
                return invokers.get(i);
            }
        }
    }
    // 如果权重相同或权重为0则均等随机
    return invokers.get(random.nextInt(length));
}
```
上面根据权重之类的来进行选择一个Invoker返回。接下来reselect的方法不在说明，是先从非selected的列表中选择，没有在从selected列表中选择。

选择好了Invoker之后，就回去FailoverClusterInvoker的doInvoke方法，接着就是根据选中的Invoker调用invoke方法进行返回结果，接着就是到具体的Invoker进行调用的过程了。这部分的解析在消费者和提供者请求响应过程已经解析过了，不再重复。

## 路由
回到AbstractClusterInvoker的invoke方法中，这里有一步是`List<Invoker<T>> invokers = list(invocation);`获取Invoker列表，这里同时也进行了路由的操作，看下list方法：

```
protected  List<Invoker<T>> list(Invocation invocation) throws RpcException {
    List<Invoker<T>> invokers = directory.list(invocation);
    return invokers;
}
```

接着看AbstractDirectory的list方法：

```
public List<Invoker<T>> list(Invocation invocation) throws RpcException {
    if (destroyed){
        throw new RpcException("Directory already destroyed .url: "+ getUrl());
    }
    //RegistryDirectory中的doList实现
    List<Invoker<T>> invokers = doList(invocation);
    List<Router> localRouters = this.routers; // local reference
    if (localRouters != null && localRouters.size() > 0) {
        for (Router router: localRouters){
            try {
                if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, true)) {
                    //路由选择
                    //MockInvokersSelector中
                    invokers = router.route(invokers, getConsumerUrl(), invocation);
                }
            } catch (Throwable t) {。。。}
        }
    }
    return invokers;
}
```
路由来过滤之后，进行负载均衡的处理。