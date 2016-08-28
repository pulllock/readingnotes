# 概览
NIO Java1.4引入，Nina基于NIO 1，Java7 设计了新的NIO 2.

`java.nio.*`包含了：

* Buffers 缓冲-数据容器
* Chartsets 字符集 -字节和Unicode码的翻译容器
* Channel 通道-表示实体I/O操作的连接
* Selector 选择器-提供可选择的，多路非阻塞IO
* Regexps 正则表达式-提供一些操作正则表达式的工具

# Mina架构

![](mina_app_arch.png)

基于Mina的应用划分为三个层次：

* I/O Service 具体的IO操作
* I/O Filter Chain 将字节过滤转换为想要的数据结构
* I/O Handler 实现实际的业务逻辑

创建一个Mina应用步骤：

1. 创建一个I/O Service 从已存在的可用service(*Acceptor)中挑选一个或者创建自己的
2. 创建一个I/O Filter Chain 从现有Filter中挑选，或者创建一个用于转换请求响应的自定义Filter
3. 创建一个I/O Handler 处理不同消息时编写的具体业务逻辑

# 服务端架构

![](Server_arch.png)

* I/O Acceptor 监听网络获取连接或包
* 新连接会创建新session，统一ip/端口号组合的请求在同一session中处理
* session中接收到的包会经过Filter Chain处理
* 最后交给IOHandler 实现各种具体业务逻辑

# 客户端架构

![](clientdiagram.png)

* 创建一个IOConnector，绑定服务器
* 连接创建，session创建并关联到该连接
* 应用或者客户端写入session，经过FilterChain后发送给服务器
* 所有接受自服务器的响应由IOHandler接受并处理

# IOService
支持所有IO服务的基类，不管是在服务端还是客户端，处理所有与应用的交互，远程对端的交互，发送并接收消息，管理session，管理连接等等。

是一个接口，服务端实现为IoAcceptor，客户端为IoConnector

> 源码版本2.0.13

```
public interface IoService {
	//获取连接通信的元数据
    TransportMetadata getTransportMetadata();

    //添加一个IoServiceListener，对IoService相关事件进行监听
    void addListener(IoServiceListener listener);

    //移除存在的监听器
    void removeListener(IoServiceListener listener);
    
    //如果dispose方法已被调用，返回true，不能说明服务已经停止，可能还有一些session正在处理中
    boolean isDisposing();
	
	//如果dispose方法已被调用，并且执行中的线程已经结束，返回true
    boolean isDisposed();

    //停止服务，服务只能在所有等待中的session都被处理后才能停止
    void dispose();

    //参数为true的话，等待每一个执行中的线程正常结束
    void dispose(boolean awaitTermination);

    //服务实例化之后获取关联的LoHandler
    IoHandler getHandler();

    //服务实例化后添加关联的IoHandler
    void setHandler(IoHandler handler);

    //获取当前服务管理的所有session
    Map<Long, IoSession> getManagedSessions();

    //获取当前服务管理的所有session总数
    int getManagedSessionCount();

    //返回当前服务创建的新的IoSession的默认配置
    IoSessionConfig getSessionConfig();

    //返回一个能够生成IoFilterChain的生成器，过滤器链被当前服务创建的IoSession使用
    IoFilterChainBuilder getFilterChainBuilder();

    //添加一个能够生成IoFilterChain的生成器，生成过滤器链被当前服务创建的IoSession使用
    void setFilterChainBuilder(IoFilterChainBuilder builder);

    //返回正在使用的过滤器链
    DefaultIoFilterChainBuilder getFilterChain();

    //如果服务能够接受连入请求返回true
    boolean isActive();

    //返回服务的活跃时间，如果服务已经不活跃，则返回最后的活跃时间
    long getActivationTime();

    //给当前服务管理的所有IoSession广播消息
    Set<WriteFuture> broadcast(Object message);

    //返回新创建的session的数据结构工厂
    //用户在session中可能需要存储一些自定义的属性，默认情况下是Map，用户可以自定义其他的数据结构
    IoSessionDataStructureFactory getSessionDataStructureFactory();
	
	//设置新创建的session的数据结构工厂
    void setSessionDataStructureFactory(IoSessionDataStructureFactory sessionDataStructureFactory);

    //返回应该被写入的字节数
    int getScheduledWriteBytes();

    //返回应该被写入的消息数
    int getScheduledWriteMessages();

    //统计功能
    IoServiceStatistics getStatistics();
}

```

AbstractIoService是抽象类，实现了IoService，对IoService进行了基础的实现，包含了一个Executor来处理接收的事件

```
public abstract class AbstractIoService implements IoService {

    private static final Logger LOGGER = LoggerFactory.getLogger(AbstractIoService.class);

    //Service id
    private static final AtomicInteger id = new AtomicInteger();

    private final String threadName;

    //处理IO的Executor
    private final Executor executor;

    //用来标志executor是在当前实例被创建的，不是调用者传过来的
    private final boolean createdExecutor;

    //负责管理所有I/O活动的IoHandler
    private IoHandler handler;

    //新创建的session使用的默认的session配置
    protected final IoSessionConfig sessionConfig;
	
	//服务监听器，服务激活时做了一些统计，其他的方法都是空的实现
    private final IoServiceListener serviceActivationListener = new IoServiceListener() {
        public void serviceActivated(IoService service) {
            
            AbstractIoService s = (AbstractIoService) service;
            IoServiceStatistics _stats = s.getStatistics();
            _stats.setLastReadTime(s.getActivationTime());
            _stats.setLastWriteTime(s.getActivationTime());
            _stats.setLastThroughputCalculationTime(s.getActivationTime());

        }

        public void serviceDeactivated(IoService service) throws Exception {}

        public void serviceIdle(IoService service, IdleStatus idleStatus) throws Exception {}

        public void sessionCreated(IoSession session) throws Exception {}

        public void sessionClosed(IoSession session) throws Exception {}

        public void sessionDestroyed(IoSession session) throws Exception {}
    };

    //默认过滤器链builder
    private IoFilterChainBuilder filterChainBuilder = new DefaultIoFilterChainBuilder();
	//默认session属性数据结构工厂
    private IoSessionDataStructureFactory sessionDataStructureFactory = new DefaultIoSessionDataStructureFactory();

     //保存当前服务的监听器
    private final IoServiceListenerSupport listeners;

     //服务销毁时的锁
    protected final Object disposalLock = new Object();
    
	//正在销毁标志
    private volatile boolean disposing;
	//已经销毁标志
    private volatile boolean disposed;
	
	//统计
    private IoServiceStatistics stats = new IoServiceStatistics(this);

    //构造函数，需要提供默认session配置和executor，如果executor为空，默认使用Executors的newCachedThreadPool()
    protected AbstractIoService(IoSessionConfig sessionConfig, Executor executor) {
        if (sessionConfig == null) {
            throw new IllegalArgumentException("sessionConfig");
        }

        if (getTransportMetadata() == null) {
            throw new IllegalArgumentException("TransportMetadata");
        }

        if (!getTransportMetadata().getSessionConfigType().isAssignableFrom(sessionConfig.getClass())) {
            throw new IllegalArgumentException("sessionConfig type: " + sessionConfig.getClass() + " (expected: "
                    + getTransportMetadata().getSessionConfigType() + ")");
        }

        //创建监听器，并添加激活监听器
        listeners = new IoServiceListenerSupport(this);
        listeners.add(serviceActivationListener);

        //默认session配置
        this.sessionConfig = sessionConfig;

        // Make JVM load the exception monitor before some transports
        // change the thread context class loader.
        ExceptionMonitor.getInstance();

        if (executor == null) {
            this.executor = Executors.newCachedThreadPool();
            createdExecutor = true;
        } else {
            this.executor = executor;
            createdExecutor = false;
        }

        threadName = getClass().getSimpleName() + '-' + id.incrementAndGet();
    }

    public final IoFilterChainBuilder getFilterChainBuilder() {
        return filterChainBuilder;
    }

    public final void setFilterChainBuilder(IoFilterChainBuilder builder) {
        if (builder == null) {
            builder = new DefaultIoFilterChainBuilder();
        }
        filterChainBuilder = builder;
    }

    public final DefaultIoFilterChainBuilder getFilterChain() {
        if (filterChainBuilder instanceof DefaultIoFilterChainBuilder) {
            return (DefaultIoFilterChainBuilder) filterChainBuilder;
        }

        throw new IllegalStateException("Current filter chain builder is not a DefaultIoFilterChainBuilder.");
    }

    public final void addListener(IoServiceListener listener) {
        listeners.add(listener);
    }

    public final void removeListener(IoServiceListener listener) {
        listeners.remove(listener);
    }

    public final boolean isActive() {
        return listeners.isActive();
    }

    public final boolean isDisposing() {
        return disposing;
    }


    public final boolean isDisposed() {
        return disposed;
    }

	//停止服务，服务只能在所有等待中的session都被处理后才能停止
    public final void dispose() {
        dispose(false);
    }
	//参数为true的话，等待每一个执行中的线程正常结束
    public final void dispose(boolean awaitTermination) {
    	//已停止，返回
        if (disposed) {
            return;
        }
		//销毁时，使用锁定
        synchronized (disposalLock) {
            if (!disposing) {
                disposing = true;

                try {
                		//调用真正停止的方法，空方法，留给子类实现
                    dispose0();
                } catch (Exception e) {
                    ExceptionMonitor.getInstance().exceptionCaught(e);
                }
            }
        }
		
		//如果是当前服务创建的Executor，调用ExecutorService的关闭方法
        if (createdExecutor) {
            ExecutorService e = (ExecutorService) executor;
            e.shutdownNow();
            //需要等待线程正常结束
            if (awaitTermination) {

                try {
                    e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                } catch (InterruptedException e1) {
                    // Restore the interrupted status
                    Thread.currentThread().interrupt();
                }
            }
        }
        //设置已被销毁标志
        disposed = true;
    }

    protected abstract void dispose0() throws Exception;

    public final Map<Long, IoSession> getManagedSessions() {
        return listeners.getManagedSessions();
    }

    public final int getManagedSessionCount() {
        return listeners.getManagedSessionCount();
    }

    public final IoHandler getHandler() {
        return handler;
    }

    public final void setHandler(IoHandler handler) {
        if (handler == null) {
            throw new IllegalArgumentException("handler cannot be null");
        }

        if (isActive()) {
            throw new IllegalStateException("handler cannot be set while the service is active.");
        }

        this.handler = handler;
    }

    public final IoSessionDataStructureFactory getSessionDataStructureFactory() {
        return sessionDataStructureFactory;
    }

    public final void setSessionDataStructureFactory(IoSessionDataStructureFactory sessionDataStructureFactory) {
        if (sessionDataStructureFactory == null) {
            throw new IllegalArgumentException("sessionDataStructureFactory");
        }

        if (isActive()) {
            throw new IllegalStateException("sessionDataStructureFactory cannot be set while the service is active.");
        }

        this.sessionDataStructureFactory = sessionDataStructureFactory;
    }

    public IoServiceStatistics getStatistics() {
        return stats;
    }

    public final long getActivationTime() {
        return listeners.getActivationTime();
    }

    /**
     * {@inheritDoc}
     */
    public final Set<WriteFuture> broadcast(Object message) {
		//IoUtil的广播方法
        final List<WriteFuture> futures = IoUtil.broadcast(message, getManagedSessions().values());
        return new AbstractSet<WriteFuture>() {
            @Override
            public Iterator<WriteFuture> iterator() {
                return futures.iterator();
            }

            @Override
            public int size() {
                return futures.size();
            }
        };
    }

    public final IoServiceListenerSupport getListeners() {
        return listeners;
    }

    protected final void executeWorker(Runnable worker) {
        executeWorker(worker, null);
    }

    protected final void executeWorker(Runnable worker, String suffix) {
        String actualThreadName = threadName;
        if (suffix != null) {
            actualThreadName = actualThreadName + '-' + suffix;
        }
        executor.execute(new NamePreservingRunnable(worker, actualThreadName));
    }

    protected final void initSession(IoSession session, IoFuture future, IoSessionInitializer sessionInitializer) {
        // Update lastIoTime if needed.
        if (stats.getLastReadTime() == 0) {
            stats.setLastReadTime(getActivationTime());
        }

        if (stats.getLastWriteTime() == 0) {
            stats.setLastWriteTime(getActivationTime());
        }      
        try {
            ((AbstractIoSession) session).setAttributeMap(session.getService().getSessionDataStructureFactory()
                    .getAttributeMap(session));
        } catch (IoSessionInitializationException e) {
            throw e;
        } catch (Exception e) {
            throw new IoSessionInitializationException("Failed to initialize an attributeMap.", e);
        }

        try {
            ((AbstractIoSession) session).setWriteRequestQueue(session.getService().getSessionDataStructureFactory()
                    .getWriteRequestQueue(session));
        } catch (IoSessionInitializationException e) {
            throw e;
        } catch (Exception e) {
            throw new IoSessionInitializationException("Failed to initialize a writeRequestQueue.", e);
        }

        if ((future != null) && (future instanceof ConnectFuture)) {
            // DefaultIoFilterChain will notify the future. (We support ConnectFuture only for now).
            session.setAttribute(DefaultIoFilterChain.SESSION_CREATED_FUTURE, future);
        }

        if (sessionInitializer != null) {
            sessionInitializer.initializeSession(session, future);
        }
		//session初始化结束时调用，空实现，留给子类实现
        finishSessionInitialization0(session, future);
    }

    protected void finishSessionInitialization0(IoSession session, IoFuture future) {

    }

    protected static class ServiceOperationFuture extends DefaultIoFuture {
        public ServiceOperationFuture() {
            super(null);
        }

        public final boolean isDone() {
            return getValue() == Boolean.TRUE;
        }

        public final void setDone() {
            setValue(Boolean.TRUE);
        }

        public final Exception getException() {
            if (getValue() instanceof Exception) {
                return (Exception) getValue();
            }

            return null;
        }

        public final void setException(Exception exception) {
            if (exception == null) {
                throw new IllegalArgumentException("exception");
            }
            
            setValue(exception);
        }
    }

    public int getScheduledWriteBytes() {
        return stats.getScheduledWriteBytes();
    }

    public int getScheduledWriteMessages() {
        return stats.getScheduledWriteMessages();
    }

}

```

## IoAcceptor
负责客户端和服务端连接的创建，服务器端接收连入的连接请求。

具体实现类：

* NioSocketAcceptor 非阻塞套接字传输 IoAcceptor
* NioDatagramAcceptor 非阻塞UDP传输 IoAcceptor
* AprSocketAcceptor 基于APR的阻塞套接字传输 IoAcceptor
* VmPipeSocketAcceptor in-VM IoAcceptor

![](IoServiceAcceptor.png)

IoAcceptor也是一个接口，继承自IoService

```
public interface IoAcceptor extends IoService {
    //返回当前绑定的本机的地址，有多个的时候，只返回一个
    SocketAddress getLocalAddress();
    Set<SocketAddress> getLocalAddresses();
    SocketAddress getDefaultLocalAddress();
    List<SocketAddress> getDefaultLocalAddresses();

    //设置本机默认地址
    void setDefaultLocalAddress(SocketAddress localAddress);
    void setDefaultLocalAddresses(SocketAddress firstLocalAddress, SocketAddress... otherLocalAddresses);
    void setDefaultLocalAddresses(Iterable<? extends SocketAddress> localAddresses);
    void setDefaultLocalAddresses(List<? extends SocketAddress> localAddresses);

    //当acceptor解除绑定本地地址时，仅当所有的客户端都关闭了才返回true
    boolean isCloseOnDeactivation();

    //设置当acceptor解除绑定本地地址时，是否所有的session都要关闭
    void setCloseOnDeactivation(boolean closeOnDeactivation);

   	//绑定地址，并且开始接受连接
    void bind() throws IOException;
    void bind(SocketAddress localAddress) throws IOException;
    void bind(SocketAddress firstLocalAddress, SocketAddress... addresses) throws IOException;
    void bind(SocketAddress... addresses) throws IOException;
    void bind(Iterable<? extends SocketAddress> localAddresses) throws IOException;

    //解除绑定，停止接受连接，所有的已经存在的连接都会被关闭。
    void unbind();
    void unbind(SocketAddress localAddress);
    void unbind(SocketAddress firstLocalAddress, SocketAddress... otherLocalAddresses);
    void unbind(Iterable<? extends SocketAddress> localAddresses);

    //为特定的地址新建session
    IoSession newSession(SocketAddress remoteAddress, SocketAddress localAddress);
}
```

### AbstractIoAcceptor

AbstractIoAcceptor是一个抽象类，对IoAcceptor做一些基本实现，继承自 AbstractIoService 实现 IoAcceptor。

```
public abstract class AbstractIoAcceptor extends AbstractIoService implements IoAcceptor {
	//默认本机地址列表
    private final List<SocketAddress> defaultLocalAddresses = new ArrayList<SocketAddress>();
	//不可修改的地址列表
    private final List<SocketAddress> unmodifiableDefaultLocalAddresses = Collections
            .unmodifiableList(defaultLocalAddresses);
	//绑定地址
    private final Set<SocketAddress> boundAddresses = new HashSet<SocketAddress>();
	//解除绑定断开连接标志
    private boolean disconnectOnUnbind = true;

    //绑定锁，绑定或者解绑的时候使用此锁
    protected final Object bindLock = new Object();

    //默认构造，跟AbstractIoService一样
    protected AbstractIoAcceptor(IoSessionConfig sessionConfig, Executor executor) {
        super(sessionConfig, executor);
        defaultLocalAddresses.add(null);
    }

    public SocketAddress getLocalAddress() {
        Set<SocketAddress> localAddresses = getLocalAddresses();
        if (localAddresses.isEmpty()) {
            return null;
        }

        return localAddresses.iterator().next();
    }

    public final Set<SocketAddress> getLocalAddresses() {
        Set<SocketAddress> localAddresses = new HashSet<SocketAddress>();

        synchronized (boundAddresses) {
            localAddresses.addAll(boundAddresses);
        }

        return localAddresses;
    }

    public SocketAddress getDefaultLocalAddress() {
        if (defaultLocalAddresses.isEmpty()) {
            return null;
        }
        return defaultLocalAddresses.iterator().next();
    }

    public final void setDefaultLocalAddress(SocketAddress localAddress) {
        setDefaultLocalAddresses(localAddress);
    }

    public final List<SocketAddress> getDefaultLocalAddresses() {
        return unmodifiableDefaultLocalAddresses;
    }

    public final void setDefaultLocalAddresses(List<? extends SocketAddress> localAddresses) {
        if (localAddresses == null) {
            throw new IllegalArgumentException("localAddresses");
        }
        setDefaultLocalAddresses((Iterable<? extends SocketAddress>) localAddresses);
    }


    public final void setDefaultLocalAddresses(Iterable<? extends SocketAddress> localAddresses) {
        if (localAddresses == null) {
            throw new IllegalArgumentException("localAddresses");
        }

        synchronized (bindLock) {
            synchronized (boundAddresses) {
                if (!boundAddresses.isEmpty()) {
                    throw new IllegalStateException("localAddress can't be set while the acceptor is bound.");
                }

                Collection<SocketAddress> newLocalAddresses = new ArrayList<SocketAddress>();

                for (SocketAddress a : localAddresses) {
                    //检查地址类型
                    checkAddressType(a);
                    newLocalAddresses.add(a);
                }

                if (newLocalAddresses.isEmpty()) {
                    throw new IllegalArgumentException("empty localAddresses");
                }

                this.defaultLocalAddresses.clear();
                this.defaultLocalAddresses.addAll(newLocalAddresses);
            }
        }
    }

    public final void setDefaultLocalAddresses(SocketAddress firstLocalAddress, SocketAddress... otherLocalAddresses) {
        if (otherLocalAddresses == null) {
            otherLocalAddresses = new SocketAddress[0];
        }

        Collection<SocketAddress> newLocalAddresses = new ArrayList<SocketAddress>(otherLocalAddresses.length + 1);

        newLocalAddresses.add(firstLocalAddress);
        for (SocketAddress a : otherLocalAddresses) {
            newLocalAddresses.add(a);
        }

        setDefaultLocalAddresses(newLocalAddresses);
    }

    public final boolean isCloseOnDeactivation() {
        return disconnectOnUnbind;
    }

    public final void setCloseOnDeactivation(boolean disconnectClientsOnUnbind) {
        this.disconnectOnUnbind = disconnectClientsOnUnbind;
    }

    public final void bind() throws IOException {
        bind(getDefaultLocalAddresses());
    }


    public final void bind(SocketAddress localAddress) throws IOException {
        if (localAddress == null) {
            throw new IllegalArgumentException("localAddress");
        }

        List<SocketAddress> localAddresses = new ArrayList<SocketAddress>(1);
        localAddresses.add(localAddress);
        bind(localAddresses);
    }

    public final void bind(SocketAddress... addresses) throws IOException {
        if ((addresses == null) || (addresses.length == 0)) {
            bind(getDefaultLocalAddresses());
            return;
        }

        List<SocketAddress> localAddresses = new ArrayList<SocketAddress>(2);

        for (SocketAddress address : addresses) {
            localAddresses.add(address);
        }

        bind(localAddresses);
    }

    public final void bind(SocketAddress firstLocalAddress, SocketAddress... addresses) throws IOException {
        if (firstLocalAddress == null) {
            bind(getDefaultLocalAddresses());
        }

        if ((addresses == null) || (addresses.length == 0)) {
            bind(getDefaultLocalAddresses());
            return;
        }

        List<SocketAddress> localAddresses = new ArrayList<SocketAddress>(2);
        localAddresses.add(firstLocalAddress);

        for (SocketAddress address : addresses) {
            localAddresses.add(address);
        }

        bind(localAddresses);
    }

    public final void bind(Iterable<? extends SocketAddress> localAddresses) throws IOException {
        if (isDisposing()) {
            throw new IllegalStateException("The Accpetor disposed is being disposed.");
        }

        if (localAddresses == null) {
            throw new IllegalArgumentException("localAddresses");
        }

        List<SocketAddress> localAddressesCopy = new ArrayList<SocketAddress>();

        for (SocketAddress a : localAddresses) {
            checkAddressType(a);
            localAddressesCopy.add(a);
        }

        if (localAddressesCopy.isEmpty()) {
            throw new IllegalArgumentException("localAddresses is empty.");
        }

        boolean activate = false;
        synchronized (bindLock) {
            synchronized (boundAddresses) {
                if (boundAddresses.isEmpty()) {
                    activate = true;
                }
            }

            if (getHandler() == null) {
                throw new IllegalStateException("handler is not set.");
            }

            try {
                Set<SocketAddress> addresses = bindInternal(localAddressesCopy);

                synchronized (boundAddresses) {
                    boundAddresses.addAll(addresses);
                }
            } catch (IOException e) {
                throw e;
            } catch (RuntimeException e) {
                throw e;
            } catch (Exception e) {
                throw new RuntimeIoException("Failed to bind to: " + getLocalAddresses(), e);
            }
        }

        if (activate) {
            getListeners().fireServiceActivated();
        }
    }

    public final void unbind() {
        unbind(getLocalAddresses());
    }

    public final void unbind(SocketAddress localAddress) {
        if (localAddress == null) {
            throw new IllegalArgumentException("localAddress");
        }

        List<SocketAddress> localAddresses = new ArrayList<SocketAddress>(1);
        localAddresses.add(localAddress);
        unbind(localAddresses);
    }

    public final void unbind(SocketAddress firstLocalAddress, SocketAddress... otherLocalAddresses) {
        if (firstLocalAddress == null) {
            throw new IllegalArgumentException("firstLocalAddress");
        }
        if (otherLocalAddresses == null) {
            throw new IllegalArgumentException("otherLocalAddresses");
        }

        List<SocketAddress> localAddresses = new ArrayList<SocketAddress>();
        localAddresses.add(firstLocalAddress);
        Collections.addAll(localAddresses, otherLocalAddresses);
        unbind(localAddresses);
    }

    public final void unbind(Iterable<? extends SocketAddress> localAddresses) {
        if (localAddresses == null) {
            throw new IllegalArgumentException("localAddresses");
        }

        boolean deactivate = false;
        synchronized (bindLock) {
            synchronized (boundAddresses) {
                if (boundAddresses.isEmpty()) {
                    return;
                }

                List<SocketAddress> localAddressesCopy = new ArrayList<SocketAddress>();
                int specifiedAddressCount = 0;

                for (SocketAddress a : localAddresses) {
                    specifiedAddressCount++;

                    if ((a != null) && boundAddresses.contains(a)) {
                        localAddressesCopy.add(a);
                    }
                }

                if (specifiedAddressCount == 0) {
                    throw new IllegalArgumentException("localAddresses is empty.");
                }

                if (!localAddressesCopy.isEmpty()) {
                    try {
                        unbind0(localAddressesCopy);
                    } catch (RuntimeException e) {
                        throw e;
                    } catch (Exception e) {
                        throw new RuntimeIoException("Failed to unbind from: " + getLocalAddresses(), e);
                    }

                    boundAddresses.removeAll(localAddressesCopy);

                    if (boundAddresses.isEmpty()) {
                        deactivate = true;
                    }
                }
            }
        }

        if (deactivate) {
            getListeners().fireServiceDeactivated();
        }
    }

    protected abstract Set<SocketAddress> bindInternal(List<? extends SocketAddress> localAddresses) throws Exception;

    protected abstract void unbind0(List<? extends SocketAddress> localAddresses) throws Exception;

    @Override
    public String toString() {
        TransportMetadata m = getTransportMetadata();
        return '('
                + m.getProviderName()
                + ' '
                + m.getName()
                + " acceptor: "
                + (isActive() ? "localAddress(es): " + getLocalAddresses() + ", managedSessionCount: "
                        + getManagedSessionCount() : "not bound") + ')';
    }

    private void checkAddressType(SocketAddress a) {
        if (a != null && !getTransportMetadata().getAddressType().isAssignableFrom(a.getClass())) {
            throw new IllegalArgumentException("localAddress type: " + a.getClass().getSimpleName() + " (expected: "
                    + getTransportMetadata().getAddressType().getSimpleName() + ")");
        }
    }

    public static class AcceptorOperationFuture extends ServiceOperationFuture {
        private final List<SocketAddress> localAddresses;

        public AcceptorOperationFuture(List<? extends SocketAddress> localAddresses) {
            this.localAddresses = new ArrayList<SocketAddress>(localAddresses);
        }

        public final List<SocketAddress> getLocalAddresses() {
            return Collections.unmodifiableList(localAddresses);
        }

        /**
         * @see Object#toString()
         */
        public String toString() {
            StringBuilder sb = new StringBuilder();

            sb.append("Acceptor operation : ");

            if (localAddresses != null) {
                boolean isFirst = true;

                for (SocketAddress address : localAddresses) {
                    if (isFirst) {
                        isFirst = false;
                    } else {
                        sb.append(", ");
                    }

                    sb.append(address);
                }
            }
            return sb.toString();
        }
    }
}
```

### AbstractPollingIoAcceptor

AbstractPollingIoAcceptor是一个抽象类，使用轮询策略的基本实现，底层sockets会一直轮询检查，有socket需要处理的时候唤醒。

```
public abstract class AbstractPollingIoAcceptor<S extends AbstractIoSession, H> extends AbstractIoAcceptor {
    //信号量，确保Selector在创建完成之前首先唤醒
    private final Semaphore lock = new Semaphore(1);

    private final IoProcessor<S> processor;

    private final boolean createdProcessor;

    private final Queue<AcceptorOperationFuture> registerQueue = new ConcurrentLinkedQueue<AcceptorOperationFuture>();

    private final Queue<AcceptorOperationFuture> cancelQueue = new ConcurrentLinkedQueue<AcceptorOperationFuture>();

    private final Map<SocketAddress, H> boundHandles = Collections.synchronizedMap(new HashMap<SocketAddress, H>());

    private final ServiceOperationFuture disposalFuture = new ServiceOperationFuture();

    //acceptor创建和初始化完成标志
    private volatile boolean selectable;

    //负责接收传入请求的线程
    private AtomicReference<Acceptor> acceptorRef = new AtomicReference<Acceptor>();

    protected boolean reuseAddress = false;

    //可以被接收的socket的数目，默认50
    protected int backlog = 50;

    protected AbstractPollingIoAcceptor(IoSessionConfig sessionConfig, Class<? extends IoProcessor<S>> processorClass) {
        this(sessionConfig, null, new SimpleIoProcessorPool<S>(processorClass), true, null);
    }
    
    protected AbstractPollingIoAcceptor(IoSessionConfig sessionConfig, Class<? extends IoProcessor<S>> processorClass,
            int processorCount) {
        this(sessionConfig, null, new SimpleIoProcessorPool<S>(processorClass, processorCount), true, null);
    }

    protected AbstractPollingIoAcceptor(IoSessionConfig sessionConfig, Class<? extends IoProcessor<S>> processorClass,
            int processorCount, SelectorProvider selectorProvider ) {
        this(sessionConfig, null, new SimpleIoProcessorPool<S>(processorClass, processorCount, selectorProvider), true, selectorProvider);
    }

    protected AbstractPollingIoAcceptor(IoSessionConfig sessionConfig, IoProcessor<S> processor) {
        this(sessionConfig, null, processor, false, null);
    }

    protected AbstractPollingIoAcceptor(IoSessionConfig sessionConfig, Executor executor, IoProcessor<S> processor) {
        this(sessionConfig, executor, processor, false, null);
    }
	
    private AbstractPollingIoAcceptor(IoSessionConfig sessionConfig, Executor executor, IoProcessor<S> processor,
            boolean createdProcessor, SelectorProvider selectorProvider) {
        super(sessionConfig, executor);

        if (processor == null) {
            throw new IllegalArgumentException("processor");
        }

        this.processor = processor;
        this.createdProcessor = createdProcessor;

        try {
            //初始化selector，交给子类实现
            init(selectorProvider);

            //可接受请求标志为true
            selectable = true;
        } catch (RuntimeException e) {
            throw e;
        } catch (Exception e) {
            throw new RuntimeIoException("Failed to initialize.", e);
        } finally {
            if (!selectable) {
                try {
                		//销毁方法交给子类实现
                    destroy();
                } catch (Exception e) {
                    ExceptionMonitor.getInstance().exceptionCaught(e);
                }
            }
        }
    }

    protected abstract void init() throws Exception;
    protected abstract void init(SelectorProvider selectorProvider) throws Exception;
    protected abstract void destroy() throws Exception;

    //可接受连接的数目
    protected abstract int select() throws Exception;

    protected abstract void wakeup();
    protected abstract Iterator<H> selectedHandles();
    protected abstract H open(SocketAddress localAddress) throws Exception;
    protected abstract SocketAddress localAddress(H handle) throws Exception;
    protected abstract S accept(IoProcessor<S> processor, H handle) throws Exception;
    protected abstract void close(H handle) throws Exception;
    @Override
    protected void dispose0() throws Exception {
        unbind();
        startupAcceptor();
        wakeup();
    }
    
    @Override
    protected final Set<SocketAddress> bindInternal(List<? extends SocketAddress> localAddresses) throws Exception {
        AcceptorOperationFuture request = new AcceptorOperationFuture(localAddresses);

        registerQueue.add(request);

        startupAcceptor();

        try {
            lock.acquire();

            Thread.sleep(10);
            wakeup();
        } finally {
            lock.release();
        }

        request.awaitUninterruptibly();

        if (request.getException() != null) {
            throw request.getException();
        }

        Set<SocketAddress> newLocalAddresses = new HashSet<SocketAddress>();

        for (H handle : boundHandles.values()) {
            newLocalAddresses.add(localAddress(handle));
        }

        return newLocalAddresses;
    }

    //绑定或者解绑的时候调用，accpetor为空，创建一个新的
    private void startupAcceptor() throws InterruptedException {
        if (!selectable) {
            registerQueue.clear();
            cancelQueue.clear();
        }

        Acceptor acceptor = acceptorRef.get();

        if (acceptor == null) {
            lock.acquire();
            acceptor = new Acceptor();

            if (acceptorRef.compareAndSet(null, acceptor)) {
                executeWorker(acceptor);
            } else {
                lock.release();
            }
        }
    }

    @Override
    protected final void unbind0(List<? extends SocketAddress> localAddresses) throws Exception {
        AcceptorOperationFuture future = new AcceptorOperationFuture(localAddresses);

        cancelQueue.add(future);
        startupAcceptor();
        wakeup();

        future.awaitUninterruptibly();
        if (future.getException() != null) {
            throw future.getException();
        }
    }

    private class Acceptor implements Runnable {
        public void run() {
            assert (acceptorRef.get() == this);

            int nHandles = 0;

            // Release the lock
            lock.release();

            while (selectable) {
                try {
                    
                    int selected = select();

                    nHandles += registerHandles();
                    
                    if (nHandles == 0) {
                        acceptorRef.set(null);

                        if (registerQueue.isEmpty() && cancelQueue.isEmpty()) {
                            assert (acceptorRef.get() != this);
                            break;
                        }

                        if (!acceptorRef.compareAndSet(null, this)) {
                            assert (acceptorRef.get() != this);
                            break;
                        }

                        assert (acceptorRef.get() == this);
                    }

                    if (selected > 0) {
                        processHandles(selectedHandles());
                    }

                    nHandles -= unregisterHandles();
                } catch (ClosedSelectorException cse) {
                    ExceptionMonitor.getInstance().exceptionCaught(cse);
                    break;
                } catch (Exception e) {
                    ExceptionMonitor.getInstance().exceptionCaught(e);

                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e1) {
                        ExceptionMonitor.getInstance().exceptionCaught(e1);
                    }
                }
            }

            if (selectable && isDisposing()) {
                selectable = false;
                try {
                    if (createdProcessor) {
                        processor.dispose();
                    }
                } finally {
                    try {
                        synchronized (disposalLock) {
                            if (isDisposing()) {
                                destroy();
                            }
                        }
                    } catch (Exception e) {
                        ExceptionMonitor.getInstance().exceptionCaught(e);
                    } finally {
                        disposalFuture.setDone();
                    }
                }
            }
        }

        //处理新的session
        private void processHandles(Iterator<H> handles) throws Exception {
            while (handles.hasNext()) {
                H handle = handles.next();
                handles.remove();

                S session = accept(processor, handle);

                if (session == null) {
                    continue;
                }

                initSession(session, null, null);

                session.getProcessor().add(session);
            }
        }
    }

    //简历socket通信
    private int registerHandles() {
        for (;;) {
            AcceptorOperationFuture future = registerQueue.poll();

            if (future == null) {
                return 0;
            }

            Map<SocketAddress, H> newHandles = new ConcurrentHashMap<SocketAddress, H>();
            List<SocketAddress> localAddresses = future.getLocalAddresses();

            try {
                
                for (SocketAddress a : localAddresses) {
                    H handle = open(a);
                    newHandles.put(localAddress(handle), handle);
                }
                boundHandles.putAll(newHandles);

                future.setDone();
                return newHandles.size();
            } catch (Exception e) {
                future.setException(e);
            } finally {
                if (future.getException() != null) {
                    for (H handle : newHandles.values()) {
                        try {
                            close(handle);
                        } catch (Exception e) {
                            ExceptionMonitor.getInstance().exceptionCaught(e);
                        }
                    }

                    wakeup();
                }
            }
        }
    }

    private int unregisterHandles() {
        int cancelledHandles = 0;
        for (;;) {
            AcceptorOperationFuture future = cancelQueue.poll();
            if (future == null) {
                break;
            }

            // close the channels
            for (SocketAddress a : future.getLocalAddresses()) {
                H handle = boundHandles.remove(a);

                if (handle == null) {
                    continue;
                }

                try {
                    close(handle);
                    wakeup(); // wake up again to trigger thread death
                } catch (Exception e) {
                    ExceptionMonitor.getInstance().exceptionCaught(e);
                } finally {
                    cancelledHandles++;
                }
            }

            future.setDone();
        }

        return cancelledHandles;
    }

    public final IoSession newSession(SocketAddress remoteAddress, SocketAddress localAddress) {
        throw new UnsupportedOperationException();
    }

    public int getBacklog() {
        return backlog;
    }
    public void setBacklog(int backlog) {
        synchronized (bindLock) {
            if (isActive()) {
                throw new IllegalStateException("backlog can't be set while the acceptor is bound.");
            }

            this.backlog = backlog;
        }
    }

    public boolean isReuseAddress() {
        return reuseAddress;
    }

    public void setReuseAddress(boolean reuseAddress) {
        synchronized (bindLock) {
            if (isActive()) {
                throw new IllegalStateException("backlog can't be set while the acceptor is bound.");
            }

            this.reuseAddress = reuseAddress;
        }
    }
    public SocketSessionConfig getSessionConfig() {
        return (SocketSessionConfig)sessionConfig;
    }
}
```

### SocketAcceptor
SocketAcceptor也是一个接口，套接字传输的Acceptor，继承自IoAcceptor，处理TCP/IP协议的连接。

```
public interface SocketAcceptor extends IoAcceptor {
    //返回InetSocketAddress，覆盖IoAcceptor中的getLocalAddress方法
    InetSocketAddress getLocalAddress();
    InetSocketAddress getDefaultLocalAddress();
	//设置本机地址
    void setDefaultLocalAddress(InetSocketAddress localAddress);

    //SO_REUSEADDR是否可用
    boolean isReuseAddress();

    //设置SO_REUSEADDR可用
    void setReuseAddress(boolean reuseAddress);

    //可用来连接的数目
    int getBacklog();
    void setBacklog(int backlog);

    SocketSessionConfig getSessionConfig();
}
```

### DatagramAcceptor
DatagramAcceptor也是一个接口，UDP传输的Acceptor，继承自IoAcceptor，处理UDP/IP协议的连接。方法与SocketAcceptor类似。

```
public interface DatagramAcceptor extends IoAcceptor {
    InetSocketAddress getLocalAddress();
    InetSocketAddress getDefaultLocalAddress();
    void setDefaultLocalAddress(InetSocketAddress localAddress);

	//
    IoSessionRecycler getSessionRecycler();
    void setSessionRecycler(IoSessionRecycler sessionRecycler);

    DatagramSessionConfig getSessionConfig();
}

```

### NioSocketAcceptor

NioSocketAcceptor是一个final类，继承自AbstractPollingIoAcceptor，实现了SocketAcceptor接口。

final类不能被继承，不能被覆盖，由于它的方法不能够被覆盖，所以其地址引用和装载在编译期间完成，而不是在运行期间由JVM进行复杂的装载，因而简单和有效。


```
public final class NioSocketAcceptor extends AbstractPollingIoAcceptor<NioSession, ServerSocketChannel>
implements SocketAcceptor {

    private volatile Selector selector;
    private volatile SelectorProvider selectorProvider = null;

    public NioSocketAcceptor() {
        super(new DefaultSocketSessionConfig(), NioProcessor.class);
        ((DefaultSocketSessionConfig) getSessionConfig()).init(this);
    }

    public NioSocketAcceptor(int processorCount) {
        super(new DefaultSocketSessionConfig(), NioProcessor.class, processorCount);
        ((DefaultSocketSessionConfig) getSessionConfig()).init(this);
    }

    public NioSocketAcceptor(IoProcessor<NioSession> processor) {
        super(new DefaultSocketSessionConfig(), processor);
        ((DefaultSocketSessionConfig) getSessionConfig()).init(this);
    }

    public NioSocketAcceptor(Executor executor, IoProcessor<NioSession> processor) {
        super(new DefaultSocketSessionConfig(), executor, processor);
        ((DefaultSocketSessionConfig) getSessionConfig()).init(this);
    }
    
    public NioSocketAcceptor(int processorCount, SelectorProvider selectorProvider) {
        super(new DefaultSocketSessionConfig(), NioProcessor.class, processorCount, selectorProvider);
        ((DefaultSocketSessionConfig) getSessionConfig()).init(this);
        this.selectorProvider = selectorProvider;
    }

    @Override
    protected void init() throws Exception {
        selector = Selector.open();
    }

    @Override
    protected void init(SelectorProvider selectorProvider) throws Exception {
        this.selectorProvider = selectorProvider;

        if (selectorProvider == null) {
            selector = Selector.open();
        } else {
            selector = selectorProvider.openSelector();
        }
    }

    @Override
    protected void destroy() throws Exception {
        if (selector != null) {
            selector.close();
        }
    }

    public TransportMetadata getTransportMetadata() {
        return NioSocketSession.METADATA;
    }

    @Override
    public InetSocketAddress getLocalAddress() {
        return (InetSocketAddress) super.getLocalAddress();
    }

    @Override
    public InetSocketAddress getDefaultLocalAddress() {
        return (InetSocketAddress) super.getDefaultLocalAddress();
    }

    public void setDefaultLocalAddress(InetSocketAddress localAddress) {
        setDefaultLocalAddress((SocketAddress) localAddress);
    }

    @Override
    protected NioSession accept(IoProcessor<NioSession> processor, ServerSocketChannel handle) throws Exception {

        SelectionKey key = null;

        if (handle != null) {
            key = handle.keyFor(selector);
        }

        if ((key == null) || (!key.isValid()) || (!key.isAcceptable())) {
            return null;
        }

        // accept the connection from the client
        SocketChannel ch = handle.accept();

        if (ch == null) {
            return null;
        }

        return new NioSocketSession(this, processor, ch);
    }

    @Override
    protected ServerSocketChannel open(SocketAddress localAddress) throws Exception {
        // Creates the listening ServerSocket

        ServerSocketChannel channel = null;

        if (selectorProvider != null) {
            channel = selectorProvider.openServerSocketChannel();
        } else {
            channel = ServerSocketChannel.open();
        }

        boolean success = false;

        try {
            // This is a non blocking socket channel
            channel.configureBlocking(false);

            // Configure the server socket,
            ServerSocket socket = channel.socket();

            // Set the reuseAddress flag accordingly with the setting
            socket.setReuseAddress(isReuseAddress());

            // and bind.
            try {
                socket.bind(localAddress, getBacklog());
            } catch (IOException ioe) {
                // Add some info regarding the address we try to bind to the
                // message
                String newMessage = "Error while binding on " + localAddress + "\n" + "original message : "
                        + ioe.getMessage();
                Exception e = new IOException(newMessage);
                e.initCause(ioe.getCause());

                // And close the channel
                channel.close();

                throw e;
            }

            // Register the channel within the selector for ACCEPT event
            channel.register(selector, SelectionKey.OP_ACCEPT);
            success = true;
        } finally {
            if (!success) {
                close(channel);
            }
        }
        return channel;
    }

    /**
     * {@inheritDoc}
     */
    @Override
    protected SocketAddress localAddress(ServerSocketChannel handle) throws Exception {
        return handle.socket().getLocalSocketAddress();
    }

    /**
     * Check if we have at least one key whose corresponding channels is
     * ready for I/O operations.
     *
     * This method performs a blocking selection operation.
     * It returns only after at least one channel is selected,
     * this selector's wakeup method is invoked, or the current thread
     * is interrupted, whichever comes first.
     * 
     * @return The number of keys having their ready-operation set updated
     * @throws IOException If an I/O error occurs
     * @throws ClosedSelectorException If this selector is closed
     */
    @Override
    protected int select() throws Exception {
        return selector.select();
    }

    /**
     * {@inheritDoc}
     */
    @Override
    protected Iterator<ServerSocketChannel> selectedHandles() {
        return new ServerSocketChannelIterator(selector.selectedKeys());
    }

    /**
     * {@inheritDoc}
     */
    @Override
    protected void close(ServerSocketChannel handle) throws Exception {
        SelectionKey key = handle.keyFor(selector);

        if (key != null) {
            key.cancel();
        }

        handle.close();
    }

    /**
     * {@inheritDoc}
     */
    @Override
    protected void wakeup() {
        selector.wakeup();
    }

    /**
     * Defines an iterator for the selected-key Set returned by the
     * selector.selectedKeys(). It replaces the SelectionKey operator.
     */
    private static class ServerSocketChannelIterator implements Iterator<ServerSocketChannel> {
        /** The selected-key iterator */
        private final Iterator<SelectionKey> iterator;

        /**
         * Build a SocketChannel iterator which will return a SocketChannel instead of
         * a SelectionKey.
         * 
         * @param selectedKeys The selector selected-key set
         */
        private ServerSocketChannelIterator(Collection<SelectionKey> selectedKeys) {
            iterator = selectedKeys.iterator();
        }

        /**
         * Tells if there are more SockectChannel left in the iterator
         * @return <tt>true</tt> if there is at least one more
         * SockectChannel object to read
         */
        public boolean hasNext() {
            return iterator.hasNext();
        }

        /**
         * Get the next SocketChannel in the operator we have built from
         * the selected-key et for this selector.
         * 
         * @return The next SocketChannel in the iterator
         */
        public ServerSocketChannel next() {
            SelectionKey key = iterator.next();

            if (key.isValid() && key.isAcceptable()) {
                return (ServerSocketChannel) key.channel();
            }

            return null;
        }

        /**
         * Remove the current SocketChannel from the iterator
         */
        public void remove() {
            iterator.remove();
        }
    }
}
```

## IoConnector
具体实现类：

* NioSocketConnector 非阻塞套接字传输 IoConnector
* NioDatagramConnector 非阻塞UDP传输 IoConnector
* AprSocketConnector 基于APR的阻塞套接字传输IoConnector
* ProxyConnector 提供代理支持
* SerialConnector 用于串行传输
* VmPipeConnector in-VM IoConnector

![](IoServiceConnector.png)


# Session
当一个客户端连接到服务器，一个新的会话被创建，在客户端关掉连接之前会一直保存在内存中。

保存连接的持久信息，以及在请求处理过程中，会话的生命周期中服务器可能需要用到的任何信息。

## 会话的状态

* 已连接 会话已被创建并可用
* 闲置 会话在至少一段时间内没有处理任何请求
	
	- 读闲置 一段时间内没任何读操作
	- 写闲置 一段时间内没任何写操作
	- 同时闲置 一段时间内没读操作也没写操作

* 关闭中 会话正在关闭中 还有正在清空的消息，清理尚未结束
* 已关闭 会话已被关闭

![](session-state.png)

## 配置参数

* 接受缓冲大小
* 发送缓冲大小
* 空闲时间
* 写超时时间

## 统计
每个会话也会保持对会话处理结束的一些记录跟踪：

* 接受或发送的字节数
* 接受或发送的消息数
* 闲置状态
* 吞吐量

# Filter

IoFilter 过滤IoService和IoHandler之间的所有I/O事件和请求。

* LoggingFilter记录所有事件和请求
* ProtocolCodecFilter将一个连入的ByteBuffer转化为消息POJO
* CompressionFilter压缩所有数据
* SSLFilter添加SSL-TLS-StartTLS支持
* 更多

# Transport

* APR 传输
* 串行传输

# Handler
处理 MINA 所触发 I/O 事件，这一接口是在过滤器链最后完成的。

IoHandler 具有以下方法：

* sessionCreated
* sessionOpened
* sessionClosed
* sessionIdle
* exceptionCaught
* messageReceived
* messageSent

## sessionCreated

新的连接被创建时触发。对于 TCP 来说这是连接接受的结果，而对于 UDP 这个在接收到一个 UDP 包时产生。这一方法可以被用于初始化会话属性，并为一些特定连接执行一次性活动。

这个方法由 I/O 处理线程的环境中调用，应该以一个低耗时的方式实现，因为同一个线程要处理多个会话。

## sessionOpened

在一个连接被打开时调用。它总是在 sessionCreated 事件之后调用。如果配置了一个线程模型，这一方法将在 I/O 处理线程之外的一个线程中调用。

## sessionClosed

在会话被关闭时调用。会话清理活动比如清理信息可以在这里执行。

## sessionIdle

在会话变为闲置状态时触发。这一方法并不为基于 UDP 的传输调用。

## exceptionCaught

在用户代码或者 MINA 抛异常时调用。如果是一个 IOException 异常的话当前连接将被关闭。

## messageReceived

在一个消息被接收到时触发。这是一个应用最常发生的处理。你需要照顾到所有要碰到的消息类型

## messageSent

在消息响应被发送 (调用 IoSession.write()) 时触发。


# IoBuffer
Mina所用的一个字节缓存。代替ByteBuffer：

* ByteBuffer没有提供getter和putter方法
* ByteBuffer固定容量的特性，很难写入可变长度的数据

IoBuffer是抽象类，不能直接被实例化。实现了Comparable接口。

