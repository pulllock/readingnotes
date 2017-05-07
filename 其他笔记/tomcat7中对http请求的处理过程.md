每个Server可以代表Tomcat，每个Server下面有多个Service，每个Service中包含多个Connector和一个Container，Connector用来处理和客户端的通信，然后把请求交给Container进行处理。这里就简单看下处理http请求的流程。

Tomcat启动初始化之后，会有一个线程一直在等待连接的到来，接收到新的连接之后，把请求交给相关处理器进行处理，这里等待连接到来的那个角色是一个Acceptor，是JIoEndpoint的内部类，处理连接的角色是SocketProcessor，也是JIoEndpoint的内部类。

# Acceptor
初始化的过程这里不做说明，直接开始看Acceptor，这个内部类的注释如下：一个一直监听进来的TCP/IP连接的后台线程，并交给适当的processor去处理。

```
//AbstractEndpoint.Acceptor实现了Runnable接口
protected class Acceptor extends AbstractEndpoint.Acceptor {

    @Override
    public void run() {

        int errorDelay = 0;

        // 一直循环，直到接到shutdown命令
        while (running) {

            // Loop if endpoint is paused
            while (paused && running) {
                state = AcceptorState.PAUSED;
                try {
                    Thread.sleep(50);
                } catch (InterruptedException e) {
                    // Ignore
                }
            }

            if (!running) {
                break;
            }
            state = AcceptorState.RUNNING;

            try {
                //达到了最大连接数，等待
                countUpOrAwaitConnection();

                Socket socket = null;
                try {
                    //接受下一个进来的连接
                    socket = serverSocketFactory.acceptSocket(serverSocket);
                } catch (IOException ioe) {
                    //异常，连接数减一个
                    countDownConnection();
                    // Introduce delay if necessary
                    errorDelay = handleExceptionWithDelay(errorDelay);
                    throw ioe;
                }
                // Successful accept, reset the error delay
                errorDelay = 0;

                // 配置socket
                if (running && !paused && setSocketOptions(socket)) {
                    //交给合适的处理器处理Socket
                    if (!processSocket(socket)) {
                        //处理完之后，连接数减少
                        countDownConnection();
                        //处理完之后，关闭Socket
                        closeSocket(socket);
                    }
                } else {
                    countDownConnection();
                    // Close socket right away
                    closeSocket(socket);
                }
            } catch (IOException x) {。。。}
        }
        state = AcceptorState.ENDED;
    }
}
```
# processSocket
可以看到，Acceptor中就是一直循环等待连接到来，连接到来之后，获取到Socket并交给处理器去处理，下面看看processSocket的处理过程。

```
protected boolean processSocket(Socket socket) {
    //处理当前Socket中的request
    try {
        //将Socket封装成SocketWrapper
        SocketWrapper<Socket> wrapper = new SocketWrapper<Socket>(socket);
        //设置连接保持时间
        wrapper.setKeepAliveLeft(getMaxKeepAliveRequests());
        //设置是否ssl可用
        wrapper.setSecure(isSSLEnabled());
        // During shutdown, executor may be null - avoid NPE
        if (!running) {
            return false;
        }
        //创建一个SocketProcessor实例，并提交到线程池执行
        getExecutor().execute(new SocketProcessor(wrapper));
    } catch (RejectedExecutionException x) {。。。 }
    return true;
}
```

processSocket是将Socket封装成一个SocketWrapper，然后设置其他属性，以wrapper新建一个SocketProcessor实例执行。

# SocketProcessor

SocketProcessor处理器进行处理包装后的Socket，SocketProcessor也是JIoEndpoint的内部类，代码如下：

```
//实现了Runnable接口
protected class SocketProcessor implements Runnable {

    protected SocketWrapper<Socket> socket = null;
    protected SocketStatus status = null;

    public SocketProcessor(SocketWrapper<Socket> socket) {
        if (socket==null) throw new NullPointerException();
        this.socket = socket;
    }

    public SocketProcessor(SocketWrapper<Socket> socket, SocketStatus status) {
        this(socket);
        this.status = status;
    }

    @Override
    public void run() {
        boolean launch = false;
        synchronized (socket) {
            try {
                SocketState state = SocketState.OPEN;

                try {
                    // SSL 握手
                    serverSocketFactory.handshake(socket.getSocket());
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    state = SocketState.CLOSED;
                }

                if ((state != SocketState.CLOSED)) {
                    if (status == null) {
                        //调用handler的process进行处理，实现是在AbstractProtocol中
                        state = handler.process(socket, SocketStatus.OPEN_READ);
                    } else {
                        state = handler.process(socket,status);
                    }
                }
                if (state == SocketState.CLOSED) {
                    //关闭
                    countDownConnection();
                    try {
                        socket.getSocket().close();
                    } catch (IOException e) {
                        // Ignore
                    }
                } else if (state == SocketState.OPEN ||
                        state == SocketState.UPGRADING ||
                        state == SocketState.UPGRADING_TOMCAT  ||
                        state == SocketState.UPGRADED){
                    //保持连接设置为true
                    socket.setKeptAlive(true);
                    socket.access();
                    launch = true;
                } else if (state == SocketState.LONG) {
                    //作为长连接
                    socket.access();
                    waitingRequests.add(socket);
                }
            } finally {
                //launch为true表示上面状态为SocketState.OPEN
                //保持连接，继续提交到线程池执行
                if (launch) {
                    try {
                        getExecutor().execute(new SocketProcessor(socket, SocketStatus.OPEN_READ));
                    } catch (RejectedExecutionException x) {
                        try {
                            //unable to handle connection at this time
                            handler.process(socket, SocketStatus.DISCONNECT);
                        } finally {
                            countDownConnection();
                        }


                    } catch (NullPointerException npe) {。。。}
                }
            }
        }
        socket = null;
        // Finish up this request
    }

}
```
## process方法

这里主要的是调用handler的process方法处理请求，实现是在AbstractProtocol的内部抽象类AbstractConnectionHandler中：

```
public SocketState process(SocketWrapper<S> wrapper,
        SocketStatus status) {
    //Socket已经被关闭
    if (wrapper == null) {
        return SocketState.CLOSED;
    }
    //取得要处理的Socket
    S socket = wrapper.getSocket();
    if (socket == null) {
        return SocketState.CLOSED;
    }
    //从缓存中获取Socket对应的处理器
    Processor<S> processor = connections.get(socket);
    if (status == SocketStatus.DISCONNECT && processor == null) {
        return SocketState.CLOSED;
    }

    wrapper.setAsync(false);
    ContainerThreadMarker.markAsContainerThread();

    try {
        if (processor == null) {
            //从可循环使用的处理器中查找
            processor = recycledProcessors.poll();
        }
        if (processor == null) {
            //创建处理器
            //该方法在子类中实现
            //我们这里是http请求，默认是在Http11Protocol的Http11ConnectionHandler中实现的
            processor = createProcessor();
        }
        //初始化ssl的信息
        //也是在子类中实现的
        //默认是在Http11Protocol的Http11ConnectionHandler中实现的
        initSsl(wrapper, processor);

        SocketState state = SocketState.CLOSED;
        do {
            if (status == SocketStatus.DISCONNECT &&!processor.isComet()) {
            } else if (processor.isAsync() || state == SocketState.ASYNC_END) {
                //异步
                state = processor.asyncDispatch(status);
                if (state == SocketState.OPEN) {
                    getProtocol().endpoint.removeWaitingRequest(wrapper);
                    state = processor.process(wrapper);
                }
            } else if (processor.isComet()) {//往下是同步
                state = processor.event(status);
            } else if (processor.getUpgradeInbound() != null) {
                state = processor.upgradeDispatch();
            } else if (processor.isUpgrade()) {
                state = processor.upgradeDispatch(status);
            } else {
                state = processor.process(wrapper);
            }

            if (state != SocketState.CLOSED && processor.isAsync()) {
                state = processor.asyncPostProcess();
            }

            if (state == SocketState.UPGRADING) {
                HttpUpgradeHandler httpUpgradeHandler =
                        processor.getHttpUpgradeHandler();
                release(wrapper, processor, false, false);
                processor = createUpgradeProcessor(
                        wrapper, httpUpgradeHandler);
                wrapper.setUpgraded(true);
                connections.put(socket, processor);
                httpUpgradeHandler.init((WebConnection) processor);
            } else if (state == SocketState.UPGRADING_TOMCAT) {
                org.apache.coyote.http11.upgrade.UpgradeInbound inbound =
                        processor.getUpgradeInbound();
                release(wrapper, processor, false, false);
                processor = createUpgradeProcessor(wrapper, inbound);
                inbound.onUpgradeComplete();
            }
        } while (state == SocketState.ASYNC_END ||
                state == SocketState.UPGRADING ||
                state == SocketState.UPGRADING_TOMCAT);
        //长连接
        if (state == SocketState.LONG) {
            //长连接放入connections中缓存
            connections.put(socket, processor);
            longPoll(wrapper, processor);
        } else if (state == SocketState.OPEN) {
            connections.remove(socket);
            release(wrapper, processor, false, true);
        } else if (state == SocketState.SENDFILE) {
            connections.put(socket, processor);
        } else if (state == SocketState.UPGRADED) {
            connections.put(socket, processor);
            if (status != SocketStatus.OPEN_WRITE) {
                longPoll(wrapper, processor);
            }
        } else {
            connections.remove(socket);
            if (processor.isUpgrade()) {
                processor.getHttpUpgradeHandler().destroy();
            } else if (processor instanceof org.apache.coyote.http11.upgrade.UpgradeProcessor) {
                // NO-OP
            } else {
                release(wrapper, processor, true, false);
            }
        }
        return state;
    } catch(java.net.SocketException e) {。。。}

    connections.remove(socket);
    if (!(processor instanceof org.apache.coyote.http11.upgrade.UpgradeProcessor)
            && !processor.isUpgrade()) {
        release(wrapper, processor, true, false);
    }
    return SocketState.CLOSED;
}
```
## 同步的process方法

createProcessor方法不列出，就是新建一个Http11Processor实例，设置属性。上面process进行处理分为异步和同步，这里我们没有配置，默认使用同步进行处理，process实现在AbstractHttp11Processor中：

```
public SocketState process(SocketWrapper<S> socketWrapper)
    throws IOException {
    //得到请求的信息
    RequestInfo rp = request.getRequestProcessor();
    rp.setStage(org.apache.coyote.Constants.STAGE_PARSE);

    setSocketWrapper(socketWrapper);
    //输入流
    getInputBuffer().init(socketWrapper, endpoint);
    //输出流
    getOutputBuffer().init(socketWrapper, endpoint);

    keepAlive = true;
    comet = false;
    openSocket = false;
    sendfileInProgress = false;
    readComplete = true;
    if (endpoint.getUsePolling()) {
        keptAlive = false;
    } else {
        keptAlive = socketWrapper.isKeptAlive();
    }

    if (disableKeepAlive()) {
        socketWrapper.setKeepAliveLeft(0);
    }

    while (!getErrorState().isError() && keepAlive && !comet && !isAsync() &&
            upgradeInbound == null &&
            httpUpgradeHandler == null && !endpoint.isPaused()) {

        // Parsing the request header
        try {
            setRequestLineReadTimeout();
            //解析请求行
            if (!getInputBuffer().parseRequestLine(keptAlive)) {
                if (handleIncompleteRequestLineRead()) {
                    break;
                }
            }

            if (endpoint.isPaused()) {
                // 503 - Service unavailable
                response.setStatus(503);
                setErrorState(ErrorState.CLOSE_CLEAN, null);
            } else {
                keptAlive = true;
                request.getMimeHeaders().setLimit(endpoint.getMaxHeaderCount());
                request.getCookies().setLimit(getMaxCookieCount());
                // 解析请求头
                if (!getInputBuffer().parseHeaders()) {
                    openSocket = true;
                    readComplete = false;
                    break;
                }
                if (!disableUploadTimeout) {
                    setSocketTimeout(connectionUploadTimeout);
                }
            }
        } catch (IOException e) {
            。。。
            // 400 - Bad Request
            response.setStatus(400);
            setErrorState(ErrorState.CLOSE_CLEAN, t);
            getAdapter().log(request, response, 0);
        }

        if (!getErrorState().isError()) {
            rp.setStage(org.apache.coyote.Constants.STAGE_PREPARE);
            try {
                //读取完请求头之后，需要设置请求的过滤器
                prepareRequest();
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                // 500 - Internal Server Error
                response.setStatus(500);
                setErrorState(ErrorState.CLOSE_CLEAN, t);
                getAdapter().log(request, response, 0);
            }
        }

        if (maxKeepAliveRequests == 1) {
            keepAlive = false;
        } else if (maxKeepAliveRequests > 0 &&
                socketWrapper.decrementKeepAlive() <= 0) {
            keepAlive = false;
        }

        //在Adapter中处理请求
        if (!getErrorState().isError()) {
            try {
                rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
                //CoyoteAdapter处理
                adapter.service(request, response);

                setCometTimeouts(socketWrapper);
            }  catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                // 500 - Internal Server Error
                response.setStatus(500);
                setErrorState(ErrorState.CLOSE_CLEAN, t);
                getAdapter().log(request, response, 0);
            }
        }

    }

}
```

代码比较长，主要的步骤是：

1. parseRequestLine解析请求行。
2. parseHeaders解析请求头。
3. prepareReques读取完请求头之后设置过滤器。
4. adapter.service交给Adapter进行真正的处理。

## 解析请求行
parseRequestLine方法是用来解析请求行的方法，在InternalInputBuffer中，具体代码不做解析。
## 解析请求头
parseHeaders方法是用来解析请求头的方法，在InternalInputBuffer中，具体代码不做解析。
## 设置过滤器
也暂先不做解析。

## Adapter真正的进行处理请求
真正处理请求的地方在CoyoteAdapter的service方法中：

```
public void service(org.apache.coyote.Request req,
                        org.apache.coyote.Response res)
    throws Exception {
    //创建Request和Response对象，将requ和res转换
    Request request = (Request) req.getNote(ADAPTER_NOTES);
    Response response = (Response) res.getNote(ADAPTER_NOTES);

    if (request == null) {
        // Create objects
        request = connector.createRequest();
        request.setCoyoteRequest(req);
        response = connector.createResponse();
        response.setCoyoteResponse(res);

        // Link objects
        request.setResponse(response);
        response.setRequest(request);

        // Set as notes
        req.setNote(ADAPTER_NOTES, request);
        res.setNote(ADAPTER_NOTES, response);

        // Set query string encoding
        req.getParameters().setQueryStringEncoding
            (connector.getURIEncoding());

    }

    if (connector.getXpoweredBy()) {
        response.addHeader("X-Powered-By", POWERED_BY);
    }

    boolean comet = false;
    boolean async = false;
    boolean postParseSuccess = false;

    try {
        req.getRequestProcessor().setWorkerThreadName(Thread.currentThread().getName());
        //解析请求，根据Request对象找到对应的Host，Context，Wrapper对象
        postParseSuccess = postParseRequest(req, request, res, response);
        if (postParseSuccess) {
            request.setAsyncSupported(connector.getService().getContainer().getPipeline().isAsyncSupported());
            // 调用Container进行处理
            //这就交给了Engine去处理了
            //通过Pipeline链传递给最终的Servlet去处理
            connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);

            if (request.isComet()) {
                if (!response.isClosed() && !response.isError()) {
                    if (request.getAvailable() || (request.getContentLength() > 0 && (!request.isParametersParsed()))) {
                        // Invoke a read event right away if there are available bytes
                        if (event(req, res, SocketStatus.OPEN_READ)) {
                            comet = true;
                            res.action(ActionCode.COMET_BEGIN, null);
                        } else {
                            return;
                        }
                    } else {
                        comet = true;
                        res.action(ActionCode.COMET_BEGIN, null);
                    }
                } else {
                    request.setFilterChain(null);
                }
            }

        }
        //异步
        if (request.isAsync()) {
            。。。
        } else if (!comet) {
                。。。
        }
    } catch (IOException e) {。。。} finally {。。。 }
}
```
这里主要做的事情是调用postParseRquest方法对请求进行处理，然后交给Engine去真正处理请求。

### postParseRquest

```
protected boolean postParseRequest(org.apache.coyote.Request req,
                                   Request request,
                                   org.apache.coyote.Response res,
                                   Response response)
        throws Exception {

    if (! req.scheme().isNull()) {

        request.setSecure(req.scheme().equals("https"));
    } else {
        req.scheme().setString(connector.getScheme());
        request.setSecure(connector.getSecure());
    }

    String proxyName = connector.getProxyName();
    int proxyPort = connector.getProxyPort();
    if (proxyPort != 0) {
        req.setServerPort(proxyPort);
    }
    if (proxyName != null) {
        req.serverName().setString(proxyName);
    }

    // Copy the raw URI to the decodedURI
    MessageBytes decodedURI = req.decodedURI();
    decodedURI.duplicate(req.requestURI());

    // 解析url的参数
    parsePathParameters(req, request);

    // URI解码
    try {
        req.getURLDecoder().convert(decodedURI, false);
    } catch (IOException ioe) {。。。}
    //调用normalize方法判断请求路径是否正确
    if (!normalize(req.decodedURI())) {
        res.setStatus(400);
        res.setMessage("Invalid URI");
        connector.getService().getContainer().logAccess(
                request, response, 0, true);
        return false;
    }
    //字符解码
    convertURI(decodedURI, request);
    // 查看解码之后是否正常
    if (!checkNormalize(req.decodedURI())) {
        res.setStatus(400);
        res.setMessage("Invalid URI character encoding");
        connector.getService().getContainer().logAccess(
                request, response, 0, true);
        return false;
    }

    //请求映射
    MessageBytes serverName;
    if (connector.getUseIPVHosts()) {
        serverName = req.localName();
        if (serverName.isNull()) {
            res.action(ActionCode.REQ_LOCAL_NAME_ATTRIBUTE, null);
        }
    } else {
        serverName = req.serverName();
    }
    if (request.isAsyncStarted()) {
        //TODO SERVLET3 - async
        //reset mapping data, should prolly be done elsewhere
        request.getMappingData().recycle();
    }

    String version = null;
    Context versionContext = null;
    boolean mapRequired = true;

    while (mapRequired) {
        connector.getMapper().map(serverName, decodedURI, version,
                                  request.getMappingData());
        request.setContext((Context) request.getMappingData().context);
        request.setWrapper((Wrapper) request.getMappingData().wrapper);

        if (request.getContext() == null) {
            res.setStatus(404);
            res.setMessage("Not found");
            // No context, so use host
            Host host = request.getHost();
            // Make sure there is a host (might not be during shutdown)
            if (host != null) {
                host.logAccess(request, response, 0, true);
            }
            return false;
        }

        //处理sessionId
        String sessionID;
        if (request.getServletContext().getEffectiveSessionTrackingModes()
                .contains(SessionTrackingMode.URL)) {

            // Get the session ID if there was one
            sessionID = request.getPathParameter(
                    SessionConfig.getSessionUriParamName(
                            request.getContext()));
            if (sessionID != null) {
                request.setRequestedSessionId(sessionID);
                request.setRequestedSessionURL(true);
            }
        }

        // Look for session ID in cookies and SSL session
        parseSessionCookiesId(req, request);
        parseSessionSslId(request);

        sessionID = request.getRequestedSessionId();

        mapRequired = false;
        if (version != null && request.getContext() == versionContext) {
            // We got the version that we asked for. That is it.
        } else {
            version = null;
            versionContext = null;

            Object[] contexts = request.getMappingData().contexts;
            if (contexts != null && sessionID != null) {
                for (int i = (contexts.length); i > 0; i--) {
                    Context ctxt = (Context) contexts[i - 1];
                    if (ctxt.getManager().findSession(sessionID) != null) {
                        if (!ctxt.equals(request.getMappingData().context)) {
                            version = ctxt.getWebappVersion();
                            versionContext = ctxt;
                            request.getMappingData().recycle();
                            mapRequired = true;
                            request.recycleSessionInfo();
                        }
                        break;
                    }
                }
            }
        }

        if (!mapRequired && request.getContext().getPaused()) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                // Should never happen
            }
            // Reset mapping
            request.getMappingData().recycle();
            mapRequired = true;
        }
    }

    //重定向
    MessageBytes redirectPathMB = request.getMappingData().redirectPath;
    if (!redirectPathMB.isNull()) {
        String redirectPath = urlEncoder.encode(redirectPathMB.toString(), "UTF-8");
        String query = request.getQueryString();
        if (request.isRequestedSessionIdFromURL()) {
            redirectPath = redirectPath + ";" +
                    SessionConfig.getSessionUriParamName(
                        request.getContext()) +
                "=" + request.getRequestedSessionId();
        }
        if (query != null) {
            redirectPath = redirectPath + "?" + query;
        }
        response.sendRedirect(redirectPath);
        request.getContext().logAccess(request, response, 0, true);
        return false;
    }

    //过滤trace方法
    if (!connector.getAllowTrace()
            && req.method().equalsIgnoreCase("TRACE")) {
        Wrapper wrapper = request.getWrapper();
        String header = null;
        if (wrapper != null) {
            String[] methods = wrapper.getServletMethods();
            if (methods != null) {
                for (int i=0; i<methods.length; i++) {
                    if ("TRACE".equals(methods[i])) {
                        continue;
                    }
                    if (header == null) {
                        header = methods[i];
                    } else {
                        header += ", " + methods[i];
                    }
                }
            }
        }
        res.setStatus(405);
        res.addHeader("Allow", header);
        res.setMessage("TRACE method is not allowed");
        request.getContext().logAccess(request, response, 0, true);
        return false;
    }

    doConnectorAuthenticationAuthorization(req, request);

    return true;
}
```

### Engine去真正处理请求
`connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);`这句代码是Engine处理请求的地方。这里面还有复杂的过程，最后会调用Servlet去处理。这里是责任链模式的使用。

在看一下处理过程之前，先看下这里使用的责任链模式，会更容易理解。

### Container中责任链模式的使用
在责任链模式中有两个角色：抽象处理者Handler和具体的处理者ConcreteHandler。在抽象处理者中一般会定义一个处理请求的接口，另外还会包含调用下一个或者上一个处理者的方法。

在tomcat中Container就是责任链模式中的抽象处理者，StandardEngine，StandardHost，StandardContext等等是具体的处理者。

请求过来时候，Engine首先接受请求，然后传递给Host容器，接着传递给Context容器，再传递给Wrapper容器，最后给Servlet处理。

而两个容器之间进行请求传递的时候涉及到另外两个概念：Pipeline和Value。Pipeline作为请求传递的管道，这个管道连接两个处理者，Value是管道上对请求加工的组件，就相当于管道上的一个口子，通过这个口子我们可以做一些其他事情。

### 容器间处理请求
`connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);`看这行代码，首先`connector.getService().getContainer()`获取到的是一个StandardEngine，然后调用getPipeline()，得到的是一个StandardPipeline标准的管道，接着调用getFirst()方法获取Value，这里没有设置first所以返回的是一个basic，类型是StandardEngineValve，然后调用的是StandardEngineValve的invoke方法，执行处理请求，看下StandardEngineValue的处理方法：

```
public final void invoke(Request request, Response response)
    throws IOException, ServletException {

    //从request中获取Host
    Host host = request.getHost();
    if (host == null) {
        response.sendError
            (HttpServletResponse.SC_BAD_REQUEST,
             sm.getString("standardEngine.noHost",
                          request.getServerName()));
        return;
    }
    if (request.isAsyncSupported()) {
        request.setAsyncSupported(host.getPipeline().isAsyncSupported());
    }

    //获取到Host之后，交给Host进行处理，Host就是下一个处理者
    host.getPipeline().getFirst().invoke(request, response);

}
```

得到Host之后，Engine就处理完了，该Host进行处理了，也是先获得Pipeline，得到的是StandardPipeline，然后调用getFirst获取到的是StandardHostValue，调用invoke方法：

```
public final void invoke(Request request, Response response)
    throws IOException, ServletException {

    //获取Context
    Context context = request.getContext();
    。。。

    context.getPipeline().getFirst().invoke(request, response);

    。。。
```
可以看到在Host中处理也是如此，接着是调用Context进行请求处理，到了StandardContextValue的invoke方法进行处理，接着是Wrapper进行处理，调用StandardWrapperValve的invoke方法进行处理。

在StandardWrapperValue中还有构建Filter链的过程，对于Filter的处理也是责任链模式的应用，暂先不做解析。

在Filter链的最后，执行Servlet的service方法，往下就该是Servlet的执行了，有关Servlet的执行流程不再解析。到这里大概的流程就完成了，中间很多细节没有说明，等看完整个源码之后，再做详细说明。
