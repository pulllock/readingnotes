服务提供者初始化完成之后，对外暴露Exporter。服务消费者初始化完成之后，得到的是Proxy代理，方法调用的时候就是调用代理。

服务消费者经过初始化之后，得到的是一个动态代理类，InvokerInvocationHandler，包含MockClusterInvoker，MockClusterInvoker包含一个RegistryDirectory和FailoverClusterInvoker。

Java动态代理，每一个动态代理类都必须要实现InvocationHandler这个接口，并且每一个代理类的实例都关联到了一个handler，当我们通过代理对象调用一个方法的时候，这个方法就会被转发为由实现了InvocationHandler这个接口的类的invoke方法来进行调用。

# 服务消费者发起调用请求

InvokerInvocationHandler实现了InvocationHandler接口，当我们调用`helloService.sayHello();`的时候，实际上会调用invoke()方法：

```
//proxy是代理的真实对象
//method调用真实对象的方法
//args调用真实对象的方法的参数
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	//方法名sayHello
    String methodName = method.getName();
    //参数类型
    Class<?>[] parameterTypes = method.getParameterTypes();
    if (method.getDeclaringClass() == Object.class) {
        return method.invoke(invoker, args);
    }
    if ("toString".equals(methodName) && parameterTypes.length == 0) {
        return invoker.toString();
    }
    if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
        return invoker.hashCode();
    }
    if ("equals".equals(methodName) && parameterTypes.length == 1) {
        return invoker.equals(args[0]);
    }
    //invoker是MockClusterInvoker
    //首先new RpcInvocation
    //然后invoker.invoke
    //最后recreate
    //返回结果
    return invoker.invoke(new RpcInvocation(method, args)).recreate();
}
```

先看下`new RpcInvocation`，Invocation是会话域，它持有调用过程中的变量，比如方法名，参数类型等。

接着是invoker.invoke()，这里invoker是MockClusterInvoker，进入MockClusterInvoker.invoker()：

```
public Result invoke(Invocation invocation) throws RpcException {
    Result result = null;
	//获取mock属性的值，我们没有配置，默认false
    String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), Constants.MOCK_KEY, Boolean.FALSE.toString()).trim(); 
    if (value.length() == 0 || value.equalsIgnoreCase("false")){
        //这里invoker是FailoverClusterInvoker
        result = this.invoker.invoke(invocation);
    } else if (value.startsWith("force")) {
        //force:direct mock
        result = doMockInvoke(invocation, null);
    } else {
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

`result = this.invoker.invoke(invocation);`这里invoker是FailoverClusterInvoker，会首先进入AbstractClusterInvoker的invoke方法：

```
public Result invoke(final Invocation invocation) throws RpcException {
	//检查是否被销毁
    checkWheatherDestoried();
    LoadBalance loadbalance;
	//根据invocation中的参数来获取所有的invoker列表
    List<Invoker<T>> invokers = list(invocation);
    if (invokers != null && invokers.size() > 0) {
    	//我们没有配置负载均衡的参数，默认使用random
        //这里得到的是RandomLoadBalance
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                .getMethodParameter(invocation.getMethodName(),Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
    } else {
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(Constants.DEFAULT_LOADBALANCE);
    }
    //如果是异步操作默认添加invocation id
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    //这里是子类实现，FailoverClusterInvoker中，执行调用
    return doInvoke(invocation, invokers, loadbalance);
}
```

FailoverClusterInvoker.doInvoke()：

```
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    List<Invoker<T>> copyinvokers = invokers;
    //检查invokers是否为空
    checkInvokers(copyinvokers, invocation);
    //重试次数
    int len = getUrl().getMethodParameter(invocation.getMethodName(), Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
    if (len <= 0) {
        len = 1;
    }
    // retry loop.
    RpcException le = null; // last exception.
    //已经调用过的invoker
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
        //使用负载均衡选择invoker.（负载均衡咱先不做解释）
        Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
        invoked.add(invoker);
        //添加到以调用过的列表中
        RpcContext.getContext().setInvokers((List)invoked);
        try {
        	//开始调用，返回结果
            Result result = invoker.invoke(invocation);
            return result;
        } catch (RpcException e) {。。。 } finally {
            providers.add(invoker.getUrl().getAddress());
        }
    }
    throw new RpcException(。。。);
}
```
`Result result = invoker.invoke(invocation);`调用并返回结果，会首先进入InvokerWrapper，然后进入ListenerInvokerWrapper的invoke方法，接着进入AbstractInvoker的invoke：

```
public Result invoke(Invocation inv) throws RpcException {
    if(destroyed) {
        throw new RpcException(。。。);
    }
    //转成RpcInvocation
    RpcInvocation invocation = (RpcInvocation) inv;
    invocation.setInvoker(this);
    if (attachment != null && attachment.size() > 0) {
        invocation.addAttachmentsIfAbsent(attachment);
    }
    Map<String, String> context = RpcContext.getContext().getAttachments();
    if (context != null) {
        invocation.addAttachmentsIfAbsent(context);
    }
    if (getUrl().getMethodParameter(invocation.getMethodName(), Constants.ASYNC_KEY, false)){
        invocation.setAttachment(Constants.ASYNC_KEY, Boolean.TRUE.toString());
    }
    //异步的话，需要添加id
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    try {
    	//这里是DubboInvoker
        return doInvoke(invocation);
    } catch (InvocationTargetException e) { } 
}
```

DubboInvoker.doInvoke()：

```
protected Result doInvoke(final Invocation invocation) throws Throwable {
    RpcInvocation inv = (RpcInvocation) invocation;
    final String methodName = RpcUtils.getMethodName(invocation);
    inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
    inv.setAttachment(Constants.VERSION_KEY, version);

    ExchangeClient currentClient;
    //在初始化的时候，引用服务的过程中会保存一个连接到服务端的Client
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        currentClient = clients[index.getAndIncrement() % clients.length];
    }
    try {
    	//异步标志
        boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
        //单向标志
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
        int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY,Constants.DEFAULT_TIMEOUT);
        //单向的，反送完不管结果
        if (isOneway) {
            boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
            currentClient.send(inv, isSent);
            RpcContext.getContext().setFuture(null);
            return new RpcResult();
        } else if (isAsync) {//异步的，发送完需要得到Future
            ResponseFuture future = currentClient.request(inv, timeout) ;
            RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
            return new RpcResult();
        } else {//同步调用，我们这里使用的这种方式
            RpcContext.getContext().setFuture(null);
            //HeaderExchangeClient
            return (Result) currentClient.request(inv, timeout).get();
        }
    } catch (TimeoutException e) {。。。}
}
```

我们这里使用的是同步调用，看`(Result) currentClient.request(inv, timeout).get();`方法，这里的client是ReferenceCountExchangeClient，直接调用HeaderExchangeClient的request方法：

```
public ResponseFuture request(Object request, int timeout) throws RemotingException {
	//这里的Channel是HeaderExchangeChannel
    return channel.request(request, timeout);
}
```

进入HeaderExchangeChannel的request方法：

```
public ResponseFuture request(Object request, int timeout) throws RemotingException {
    if (closed) {
        throw new RemotingException(。。。);
    }
    //创建一个请求头
    Request req = new Request();
    req.setVersion("2.0.0");
    req.setTwoWay(true);
    //这里request参数里面保存着
    //methodName = "sayHello"
	//parameterTypes = {Class[0]@2814} 
	//arguments = {Object[0]@2768} 
	//attachments = {HashMap@2822}  size = 4
	//invoker = {DubboInvoker@2658}
    req.setData(request);
    DefaultFuture future = new DefaultFuture(channel, req, timeout);
    try{
    	//这里的channel是NettyClient
        //发送请求
        channel.send(req);
    }catch (RemotingException e) {
        future.cancel();
        throw e;
    }
    return future;
}
```

`channel.send(req)`，首先会调用AbstractPeer的send方法：

```
//子类处理，接着是AbstractClient执行发送
public void send(Object message) throws RemotingException {
    send(message, url.getParameter(Constants.SENT_KEY, false));
}
```

AbstractClient执行发送：

```
public void send(Object message, boolean sent) throws RemotingException {
	//重连
    if (send_reconnect && !isConnected()){
        connect();
    }
    //先获取Channel，是在NettyClient中实现的
    Channel channel = getChannel();
    //TODO getChannel返回的状态是否包含null需要改进
    if (channel == null || ! channel.isConnected()) {
      throw new RemotingException(this, "message can not send, because channel is closed . url:" + getUrl());
    }
    channel是NettyChannel
    channel.send(message, sent);
}
```

`channel.send(message, sent);`首先经过AbstractChannel的send方法处理，只是判断是否关闭了，然后是NettyChannel的send来继续处理，这里就把消息发送到服务端了：

```
public void send(Object message, boolean sent) throws RemotingException {
    super.send(message, sent);

    boolean success = true;
    int timeout = 0;
    try {
    	//交给netty处理
        ChannelFuture future = channel.write(message);
        if (sent) {
            timeout = getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
            success = future.await(timeout);
        }
        Throwable cause = future.getCause();
        if (cause != null) {
            throw cause;
        }
    } catch (Throwable e) {
        throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
    }

    if(! success) {
        throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress()
                + "in timeout(" + timeout + "ms) limit");
    }
}
```

# 服务提供者处理并响应请求

服务端已经打开端口并监听请求的到来，当服务消费者发送调用请求的时候，经过Netty的处理后会到dubbo中的codec相关方法中先进行解码，入口是`NettyCodecAdapter.messageReceived()`，关于这个方法的代码在dubbo编解码的那篇文章中已经分析过，不再重复。经过解码之后，会进入到NettyHandler.messageReceived()方法：

```
public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
	//获取channel
    NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
    try {
    	//这里handler是NettyServer
        handler.received(channel, e.getMessage());
    } finally {
        NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
    }
}
```

接着会进入AbstractPeer的received方法：

```
public void received(Channel ch, Object msg) throws RemotingException {
    if (closed) {
        return;
    }
    //这里是MultiMessageHandler
    handler.received(ch, msg);
}
```

进入MultiMessageHandler的received方法：

```
public void received(Channel channel, Object message) throws RemotingException {
	//是多消息的话，使用多消息处理器处理
    if (message instanceof MultiMessage) {
        MultiMessage list = (MultiMessage)message;
        for(Object obj : list) {
            handler.received(channel, obj);
        }
    } else {
    	//这里是HeartbeatHandler
        handler.received(channel, message);
    }
}
```

进入HeartbeatHandler的received方法：

```
public void received(Channel channel, Object message) throws RemotingException {
    setReadTimestamp(channel);
    //心跳请求处理
    if (isHeartbeatRequest(message)) {
        Request req = (Request) message;
        if (req.isTwoWay()) {
            Response res = new Response(req.getId(), req.getVersion());
            res.setEvent(Response.HEARTBEAT_EVENT);
            channel.send(res);
        }
        return;
    }
    //心跳回应消息处理
    if (isHeartbeatResponse(message)) {
        return;
    }
    //这里是AllChannelHandler
    handler.received(channel, message);
}
```

继续进入AllChannelHandler的received方法：

```
public void received(Channel channel, Object message) throws RemotingException {
	//获取线程池执行
    ExecutorService cexecutor = getExecutorService();
    try {
    	//handler是DecodeHandler
        cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
    } catch (Throwable t) { }
}
```
这里会去启动新线程执行ChannelEventRunnable的run方法，接着去调用DecodeHandler的received方法：

```
public void received(Channel channel, Object message) throws RemotingException {
	//不清楚啥意思
    if (message instanceof Decodeable) {
        decode(message);
    }
	//解码请求类型
    if (message instanceof Request) {
        decode(((Request)message).getData());
    }
	//解码响应类型
    if (message instanceof Response) {
        decode( ((Response)message).getResult());
    }
	//解码之后到HeaderExchangeHandler中处理
    handler.received(channel, message);
}
```

解码之后到HeaderExchangeHandler的received方法：

```
public void received(Channel channel, Object message) throws RemotingException {
    channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
    ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
    try {
    	//request类型的消息
        if (message instanceof Request) {
            Request request = (Request) message;
            if (request.isEvent()) {//判断心跳还是正常请求
            	//	处理心跳
                handlerEvent(channel, request);
            } else {//正常的请求
            	//需要返回
                if (request.isTwoWay()) {
                	//处理请求，并构造响应信息
                    Response response = handleRequest(exchangeChannel, request);
                    //NettyChannel，发送响应信息
                    channel.send(response);
                } else {//不需要返回的处理
                    handler.received(exchangeChannel, request.getData());
                }
            }
        } else if (message instanceof Response) {//response类型的消息
            handleResponse(channel, (Response) message);
        } else if (message instanceof String) {
            if (isClientSide(channel)) {
                Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
            } else {//telnet类型
                String echo = handler.telnet(channel, (String) message);
                if (echo != null && echo.length() > 0) {
                    channel.send(echo);
                }
            }
        } else {
            handler.received(exchangeChannel, message);
        }
    } finally {
        HeaderExchangeChannel.removeChannelIfDisconnected(channel);
    }
}
```

先看下处理请求，并构造响应信息：

```
Response handleRequest(ExchangeChannel channel, Request req) throws RemotingException {
    Response res = new Response(req.getId(), req.getVersion());
    if (req.isBroken()) {
        Object data = req.getData();

        String msg;
        if (data == null) msg = null;
        else if (data instanceof Throwable) msg = StringUtils.toString((Throwable) data);
        else msg = data.toString();
        res.setErrorMessage("Fail to decode request due to: " + msg);
        res.setStatus(Response.BAD_REQUEST);

        return res;
    }
    // find handler by message class.
    Object msg = req.getData();
    try {
        //处理请求数据，handler是DubboProtocol中的new的一个ExchangeHandlerAdapter
        Object result = handler.reply(channel, msg);
        res.setStatus(Response.OK);
        res.setResult(result);
    } catch (Throwable e) { }
    return res;
}
```
进入DubboProtocol中的ExchangeHandlerAdapter的replay方法：

```
public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
        if (message instanceof Invocation) {
        	//Invocation中保存着方法名等
            Invocation inv = (Invocation) message;
            //获取Invoker
            Invoker<?> invoker = getInvoker(channel, inv);
            //如果是callback 需要处理高版本调用低版本的问题
            if (Boolean.TRUE.toString().equals(inv.getAttachments().get(IS_CALLBACK_SERVICE_INVOKE))){
                String methodsStr = invoker.getUrl().getParameters().get("methods");
                boolean hasMethod = false;
                if (methodsStr == null || methodsStr.indexOf(",") == -1){
                    hasMethod = inv.getMethodName().equals(methodsStr);
                } else {
                    String[] methods = methodsStr.split(",");
                    for (String method : methods){
                        if (inv.getMethodName().equals(method)){
                            hasMethod = true;
                            break;
                        }
                    }
                }
                if (!hasMethod){
                    return null;
                }
            }
            RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
            //执行调用，然后返回结果
            return invoker.invoke(inv);
        }
        throw new RemotingException(。。。);
    }
```

先看下getInvoker获取Invoker：

```
Invoker<?> getInvoker(Channel channel, Invocation inv) throws RemotingException{
    boolean isCallBackServiceInvoke = false;
    boolean isStubServiceInvoke = false;
    int port = channel.getLocalAddress().getPort();
    String path = inv.getAttachments().get(Constants.PATH_KEY);
    //如果是客户端的回调服务.
    isStubServiceInvoke = Boolean.TRUE.toString().equals(inv.getAttachments().get(Constants.STUB_EVENT_KEY));
    if (isStubServiceInvoke){
        port = channel.getRemoteAddress().getPort();
    }
    //callback
    isCallBackServiceInvoke = isClientSide(channel) && !isStubServiceInvoke;
    if(isCallBackServiceInvoke){
        path = inv.getAttachments().get(Constants.PATH_KEY)+"."+inv.getAttachments().get(Constants.CALLBACK_SERVICE_KEY);
        inv.getAttachments().put(IS_CALLBACK_SERVICE_INVOKE, Boolean.TRUE.toString());
    }
    String serviceKey = serviceKey(port, path, inv.getAttachments().get(Constants.VERSION_KEY), inv.getAttachments().get(Constants.GROUP_KEY));
	//从之前缓存的exporterMap中查找Exporter
    //key：dubbo.common.hello.service.HelloService:20880
    DubboExporter<?> exporter = (DubboExporter<?>) exporterMap.get(serviceKey);

    if (exporter == null)
        throw new RemotingException(。。。）;
	//得到Invoker，返回
    return exporter.getInvoker();
}
```

再看执行调用`invoker.invoke(inv);`，会先进入InvokerWrapper：

```
public Result invoke(Invocation invocation) throws RpcException {
    return invoker.invoke(invocation);
}
```

接着进入AbstractProxyInvoker：

```
public Result invoke(Invocation invocation) throws RpcException {
    try {
    	//先doInvoke
        //然后封装成结果返回
        return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
    } catch (InvocationTargetException e) {。。。}
}
```

这里的doInvoke是在JavassistProxyFactory中的AbstractProxyInvoker实例：

```
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    // TODO Wrapper类不能正确处理带$的类名
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName, 
                                  Class<?>[] parameterTypes, 
                                  Object[] arguments) throws Throwable {
                                  //这里就调用了具体的方法
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```

消息处理完后返回到HeaderExchangeHandler的received方法：

```
public void received(Channel channel, Object message) throws RemotingException {
    channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
    ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
    try {
    	//request类型的消息
        if (message instanceof Request) {
            Request request = (Request) message;
            if (request.isEvent()) {//判断心跳还是正常请求
            	//	处理心跳
                handlerEvent(channel, request);
            } else {//正常的请求
            	//需要返回
                if (request.isTwoWay()) {
                	//处理请求，并构造响应信息，这在上面已经解析过了
                    Response response = handleRequest(exchangeChannel, request);
                    //NettyChannel，发送响应信息
                    channel.send(response);
                } else {//不需要返回的处理
                    handler.received(exchangeChannel, request.getData());
                }
            }
        } else if (message instanceof Response) {//response类型的消息
            handleResponse(channel, (Response) message);
        } else if (message instanceof String) {
            if (isClientSide(channel)) {
                Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
            } else {//telnet类型
                String echo = handler.telnet(channel, (String) message);
                if (echo != null && echo.length() > 0) {
                    channel.send(echo);
                }
            }
        } else {
            handler.received(exchangeChannel, message);
        }
    } finally {
        HeaderExchangeChannel.removeChannelIfDisconnected(channel);
    }
}
```

解析完请求，构造完响应消息，就开始发送响应了,`channel.send(response);`，先经过AbstractPeer:

```
public void send(Object message) throws RemotingException {
	//NettyChannel
    send(message, url.getParameter(Constants.SENT_KEY, false));
}
```

进入NettyChannel中，进行响应消息的发送：

```
public void send(Object message, boolean sent) throws RemotingException {
	//AbstractChannel的处理
    super.send(message, sent);

    boolean success = true;
    int timeout = 0;
    try {
        ChannelFuture future = channel.write(message);
        if (sent) {
            timeout = getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
            success = future.await(timeout);
        }
        Throwable cause = future.getCause();
        if (cause != null) {
            throw cause;
        }
    } catch (Throwable e) {
        throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
    }

    if(! success) {
        throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress()
                + "in timeout(" + timeout + "ms) limit");
    }
}
```

# 消费者接受到服务端返回的响应后的处理
服务提供者端接收到消费者端的请求并处理之后，返回给消费者端，消费者这边接受响应的入口跟提供者差不多，也是NettyCodecAdapter.messageReceived()，经过解码，到NettyHandler.messageReceived()处理：

```
public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
    NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
    try {
    	//NettyClient
        handler.received(channel, e.getMessage());
    } finally {
        NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
    }
}
```

先经过AbstractPeer的received方法：

```
public void received(Channel ch, Object msg) throws RemotingException {
    if (closed) {
        return;
    }
    //MultiMessageHandler
    handler.received(ch, msg);
}
```

进入MultiMessageHandler：

```
public void received(Channel channel, Object message) throws RemotingException {
    if (message instanceof MultiMessage) {
        MultiMessage list = (MultiMessage)message;
        for(Object obj : list) {
            handler.received(channel, obj);
        }
    } else {
    	//HeartbeatHandler
        handler.received(channel, message);
    }
}
```

进入HeartbeatHandler，根据不同类型进行处理：

```
public void received(Channel channel, Object message) throws RemotingException {
    setReadTimestamp(channel);
    if (isHeartbeatRequest(message)) {
        Request req = (Request) message;
        if (req.isTwoWay()) {
            Response res = new Response(req.getId(), req.getVersion());
            res.setEvent(Response.HEARTBEAT_EVENT);
            channel.send(res);
            if (logger.isInfoEnabled()) {
                int heartbeat = channel.getUrl().getParameter(Constants.HEARTBEAT_KEY, 0);
                if(logger.isDebugEnabled()) {
                    logger.debug("Received heartbeat from remote channel " + channel.getRemoteAddress()
                                    + ", cause: The channel has no data-transmission exceeds a heartbeat period"
                                    + (heartbeat > 0 ? ": " + heartbeat + "ms" : ""));
                }
            }
        }
        return;
    }
    if (isHeartbeatResponse(message)) {
        if (logger.isDebugEnabled()) {
            logger.debug(
                new StringBuilder(32)
                    .append("Receive heartbeat response in thread ")
                    .append(Thread.currentThread().getName())
                    .toString());
        }
        return;
    }
    //AllChannelHandler
    handler.received(channel, message);
}
```

进入AllChannelHandler：

```
public void received(Channel channel, Object message) throws RemotingException {
    ExecutorService cexecutor = getExecutorService();
    try {
        cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
    } catch (Throwable t) {
        throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
    }
}
```

然后在新线程，ChannelEventRunnable的run方法中进入DecodeHandler：

```
public void received(Channel channel, Object message) throws RemotingException {
    if (message instanceof Decodeable) {
        decode(message);
    }

    if (message instanceof Request) {
        decode(((Request)message).getData());
    }
	//这里进行response类型的处理
    if (message instanceof Response) {
        decode( ((Response)message).getResult());
    }

    handler.received(channel, message);
}
```

进入处理response的decode方法，进行解码response：

```
private void decode(Object message) {
    if (message != null && message instanceof Decodeable) {
        try {
            ((Decodeable)message).decode();
        } catch (Throwable e) {。。。} // ~ end of catch
    } // ~ end of if
} 
```

接着会进入HeaderExchangerHandler.received () 方法：

```
public void received(Channel channel, Object message) throws RemotingException {
    channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
    ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
    try {
        if (message instanceof Request) {
            Request request = (Request) message;
            if (request.isEvent()) {
                handlerEvent(channel, request);
            } else {
                if (request.isTwoWay()) {
                    Response response = handleRequest(exchangeChannel, request);
                    channel.send(response);
                } else {
                    handler.received(exchangeChannel, request.getData());
                }
            }
        } else if (message instanceof Response) {
        	//这里处理response消息
            handleResponse(channel, (Response) message);
        } else if (message instanceof String) {
            if (isClientSide(channel)) {  Exception  } else {
                String echo = handler.telnet(channel, (String) message);
                if (echo != null && echo.length() > 0) {
                    channel.send(echo);
                }
            }
        } else {
            handler.received(exchangeChannel, message);
        }
    } finally {
        HeaderExchangeChannel.removeChannelIfDisconnected(channel);
    }
}
```

handleResponse方法：

```
static void handleResponse(Channel channel, Response response) throws RemotingException {
    if (response != null && !response.isHeartbeat()) {
        DefaultFuture.received(channel, response);
    }
}
```

这一步设置response到消费者请求的Future中，以供消费者通过DefaultFuture.get()取得提供者的响应，此为同步转异步重要一步，且请求超时也由DefaultFuture控制。

然后就是`return (Result) currentClient.request(inv, timeout).get();`在DubboInvoker中，这里继续执行，然后执行Filter，最后返回到InvokerInvocationHandler.invoker()方法中，方法得到调用结果，结束！

注意：

消费者端的DubboInvoker发起请求后，后续的逻辑是异步的或是指定超时时间内阻塞的，直到得到响应结果后，继续执行DubboInvoker中逻辑。

对于异步请求时，消费者得到Future，其余逻辑均是异步的。

消费者还可以通过设置async、sent、return来调整处理逻辑，async指异步还是同步请求，sent指是否等待请求消息发出即阻塞等待是否成功发出请求、return指是否忽略返回值即但方向通信，一般异步时使用以减少Future对象的创建和管理成本。