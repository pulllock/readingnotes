主要介绍下tomcat7的启动流程，以及相关源码的分析。这里从我们常用的tomcat7的启动脚本为分析入口，然后进入到tomcat7相关源码中去。使用到的tomcat的版本为tomcat 7.0.77。

在正式进入源码分析之前，首先需要了解下tomcat的类加载器的东西。请参考[tomcat7类加载器解析](http://cxis.me/2017/05/10/tomcat7%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E8%A7%A3%E6%9E%90/)

# 启动脚本
## startup.sh
平时启动tomcat都是从这里开始，先看下这里都做了什么，下面是startup.sh源码：

```
# CATALINA服务启动脚本
。。。
# 执行的脚本是catalina.sh
EXECUTABLE=catalina.sh

。。。
# 执行catalina.sh脚本，参数是start
exec "$PRGDIR"/"$EXECUTABLE" start "$@"
```

## catalina.sh
这里脚本有点长，我们只看有关start的部分，其他的都暂先省略掉。

```
算了，这里不装逼了，脚本基本上内容都不太了解。不解析了，直接看最主要的一句
org.apache.catalina.startup.Bootstrap "$@" start \
```
上面这句就是要调用`org.apache.catalina.startup.Bootstrap`的main方法，参数是start，这里就进入了我们的源码。

# Bootstrap类
`org.apache.catalina.startup.BootStrap`，BootStrap类中的main方法是我们要分析的入口，是通过脚本启动tomcat的主方法和入口。

```
public static void main(String args[]) {

    if (daemon == null) {
        // Don't set daemon until init() has completed
        Bootstrap bootstrap = new Bootstrap();
        try {
            //初始化
            bootstrap.init();
        } catch (Throwable t) {。。。}
        daemon = bootstrap;
    } else {
        Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
    }

    try {
        String command = "start";
        if (args.length > 0) {
            command = args[args.length - 1];
        }
        //暂不知是啥意思
        if (command.equals("startd")) {
            args[args.length - 1] = "start";
            daemon.load(args);
            daemon.start();
        } else if (command.equals("stopd")) {
            args[args.length - 1] = "stop";
            daemon.stop();
        } else if (command.equals("start")) {
            //start命令
            daemon.setAwait(true);
            daemon.load(args);
            daemon.start();
        } else if (command.equals("stop")) {
            //stop命令
            daemon.stopServer(args);
        } else if (command.equals("configtest")) {
            daemon.load(args);
            if (null==daemon.getServer()) {
                System.exit(1);
            }
            System.exit(0);
        } else {。。。}
    } catch (Throwable t) {
        。。。
        System.exit(1);
    }

}
```
main方法做的事情也不复杂，当我们第一次启动的时候，首先初始化，然后调用start方法。

# 初始化
初始化的时候做的工作也不复杂，大概如下：

- 首先设置catalina home和catalina base目录。
- 初始化类加载器，这是很重要的步骤
- 设置上下文类加载器和安全相关类加载器
- 加载启动类，创建启动类的实例
- 设置启动类实例的parentClassLoader为sharedLoader

初始化BootStrap方法代码如下：

```
public void init() throws Exception {

    //设置catalina.home属性，如果不存在就使用当前工作目录
    setCatalinaHome();
    // 设置catalina.base属性，如果不存在就使用当前工作目录
    setCatalinaBase();
    //初始化类加载器
    initClassLoaders();
    //设置当前线程的类加载器
    Thread.currentThread().setContextClassLoader(catalinaLoader);

    SecurityClassLoad.securityClassLoad(catalinaLoader);

    // 使用catalinaLoader类加载器加载Catalina类
    Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
    //创建启动类的实例
    Object startupInstance = startupClass.newInstance();

    //以下设置启动类实例的parentClassLoader为sharedLoader
    String methodName = "setParentClassLoader";
    Class<?> paramTypes[] = new Class[1];
    paramTypes[0] = Class.forName("java.lang.ClassLoader");
    Object paramValues[] = new Object[1];
    paramValues[0] = sharedLoader;
    Method method =
        startupInstance.getClass().getMethod(methodName, paramTypes);
    method.invoke(startupInstance, paramValues);
    //启动实例
    catalinaDaemon = startupInstance;

}
```

## 初始化类加载器
重点看下有关类加载器的初始化，代码如下：

```
private void initClassLoaders() {
    try {
        //创建CommonClassLoader，common对应的是配置文件中的common.loader
        //这里和下面的配置文件指的是conf/catalina.properties文件
        commonLoader = createClassLoader("common", null);
        if( commonLoader == null ) {
            //配置文件中没有配置common，就使用当前类的ClassLoader
            commonLoader=this.getClass().getClassLoader();
        }
        //创建ServerClassLoader，对应配置文件中的server.loader
        catalinaLoader = createClassLoader("server", commonLoader);
        //创建SharedClassLoader，对应配置文件中的shared.loader
        sharedLoader = createClassLoader("shared", commonLoader);
    } catch (Throwable t) {。。。}
}
```
这里会创建三个ClassLoader，commonLoader，catalinaLoader，sharedLoader，配置文件`conf/catalina.properties`中`server.loader`和`shared.loader`是空的，所以在运行时这两个Loader和commonLoader是一样的，这一点可以在下面的源码中看到。目前只有commonLoader具备实际的意义。

看下createClassLoader方法：

```
private ClassLoader createClassLoader(String name, ClassLoader parent)
    throws Exception {
    //CatalinaProperties对应着conf/catalina.properties文件
    //配置文件中配置：common.loader=${catalina.base}/lib,${catalina.base}/lib/*.jar,${catalina.home}/lib,${catalina.home}/lib/*.jar
    //先从配置文件中获取name对应的name.loader
    //分别是common.loader、server.loader、shared.loader
    String value = CatalinaProperties.getProperty(name + ".loader");
    //如果value为空，就直接返回parent
    //这里也验证了上面说到的，运行时由于server.loader、shared.loader是空的，所以之前说的三个Loader其实是同一个CommonLoader
    if ((value == null) || (value.equals("")))
        return parent;
    //将value中${catalina.base}和${catalina.home}替换成实际的目录
    value = replace(value);
    //仓库列表，仓库指的是指定的目录，比如lib；或者是*.jar等
    List<Repository> repositories = new ArrayList<Repository>();
    //逗号分割
    StringTokenizer tokenizer = new StringTokenizer(value, ",");
    while (tokenizer.hasMoreElements()) {
        String repository = tokenizer.nextToken().trim();
        if (repository.length() == 0) {
            continue;
        }

        //仓库封装着对应的位置和类型，添加进list中保存
        //这里是URL类型的
        try {
            URL url = new URL(repository);
            repositories.add(
                    new Repository(repository, RepositoryType.URL));
            continue;
        } catch (MalformedURLException e) {
            // Ignore
        }

        //本地仓库
        if (repository.endsWith("*.jar")) {//多个jar
            repository = repository.substring
                (0, repository.length() - "*.jar".length());
            repositories.add(
                    new Repository(repository, RepositoryType.GLOB));
        } else if (repository.endsWith(".jar")) {//单个jar
            repositories.add(
                    new Repository(repository, RepositoryType.JAR));
        } else {//目录类型的
            repositories.add(
                    new Repository(repository, RepositoryType.DIR));
        }
    }
    //使用ClassLoaderFactory的createClassLoader方法来创建ClassLoader
    //仓库是上面解析过的仓库
    return ClassLoaderFactory.createClassLoader(repositories, parent);
}
```
继续往下看createClassLoader方法，创建新的类加载器：

```
public static ClassLoader createClassLoader(List<Repository> repositories,
                                                final ClassLoader parent)
    throws Exception {

    Set<URL> set = new LinkedHashSet<URL>();
    //将各种类型的repository解析成url，保存进set中
    if (repositories != null) {
        for (Repository repository : repositories)  {
            if (repository.getType() == RepositoryType.URL) {
                URL url = buildClassLoaderUrl(repository.getLocation());
                set.add(url);
            } else if (repository.getType() == RepositoryType.DIR) {
                File directory = new File(repository.getLocation());
                directory = directory.getCanonicalFile();
                if (!validateFile(directory, RepositoryType.DIR)) {
                    continue;
                }
                URL url = buildClassLoaderUrl(directory);
                set.add(url);
            } else if (repository.getType() == RepositoryType.JAR) {
                File file=new File(repository.getLocation());
                file = file.getCanonicalFile();
                if (!validateFile(file, RepositoryType.JAR)) {
                    continue;
                }
                URL url = buildClassLoaderUrl(file);
                set.add(url);
            } else if (repository.getType() == RepositoryType.GLOB) {
                File directory=new File(repository.getLocation());
                directory = directory.getCanonicalFile();
                if (!validateFile(directory, RepositoryType.GLOB)) {
                    continue;
                }
                String filenames[] = directory.list();
                if (filenames == null) {
                    continue;
                }
                for (int j = 0; j < filenames.length; j++) {
                    String filename = filenames[j].toLowerCase(Locale.ENGLISH);
                    if (!filename.endsWith(".jar"))
                        continue;
                    File file = new File(directory, filenames[j]);
                    file = file.getCanonicalFile();
                    if (!validateFile(file, RepositoryType.JAR)) {
                        continue;
                    };
                    URL url = buildClassLoaderUrl(file);
                    set.add(url);
                }
            }
        }
    }

    // Construct the class loader itself
    final URL[] array = set.toArray(new URL[set.size()]);
    //使用上面解析到的url来新建URLClassLoader实例
    return AccessController.doPrivileged(
            new PrivilegedAction<URLClassLoader>() {
                @Override
                public URLClassLoader run() {
                    if (parent == null)
                        return new URLClassLoader(array);
                    else
                        return new URLClassLoader(array, parent);
                }
            });
}
```
这里进行新的类加载器创建，过程并不复杂，首先根据不同类型解析仓库，然后根据解析到的url新建URLClassLoader的实例。URLClassLoader是ClassLoader的子类，用于从目录的url或者jar文件来加载类和资源。

## 创建启动类并创建新实例
上面创建完成三个ClassLoader之后，然后设置当前上下文的ClassLoader和安全ClassLoader为catalinaLoader，接下来一步就是使用catalinaLoader加载`org.apache.catalina.startup.Catalina`启动类。

```
Class<?> startupClass =
    catalinaLoader.loadClass
    ("org.apache.catalina.startup.Catalina");
Object startupInstance = startupClass.newInstance();
```

# 调用start方法启动Catalina
上面init方法完成之后，ClassLoader已经被创建，catalinaDaemon为启动的新实例。然后就该调用start方法了，调用的位置如下：

```
else if (command.equals("start")) {
    //这个是让服务器启动之后，保持运行状态，监听后面发来的命令。
   daemon.setAwait(true);
   //对tomcat的相关的配置文件进行加载解析
   //对tomcat各个组件进行初始化配置操作
   daemon.load(args);
   //启动Catalina
   daemon.start();
}
```

启动前的三个步骤如下：

- 设置await状态，让服务器启动之后，保持运行状态，监听后面发来的命令。
- load方法，对tomcat的相关的配置文件进行加载解析，并对各个组件进行初始化配置操作。
- start方法，真正启动Catalina

## load方法加载和初始化
load方法主要对tomcat的相关配置文件进行加载解析，对各个组件进行初始化配置操作，代码如下：

```
private void load(String[] arguments)
    throws Exception {

    //要调用的load()方法
    String methodName = "load";
    //启动时候参数的处理
    Object param[];
    Class<?> paramTypes[];
    if (arguments==null || arguments.length==0) {
        paramTypes = null;
        param = null;
    } else {
        paramTypes = new Class[1];
        paramTypes[0] = arguments.getClass();
        param = new Object[1];
        param[0] = arguments;
    }
    //下面调用Catalina的load方法
    Method method =
        catalinaDaemon.getClass().getMethod(methodName, paramTypes);
    method.invoke(catalinaDaemon, param);

}
```
这里没有做详细处理，只是使用反射调用Catalina实例的load方法，Catalina实例是我们上面使用反射新建的实例。接续往下看Catalina的load方法：

```
public void load() {
    //记录初始化开始的时间
    long t1 = System.nanoTime();
    //初始化目录，还有embedded的目录处理
    initDirs();

    //初始化命名空间？这里不太理解还，应该是跟JNDI有关系
    initNaming();

    //创建并配置一个Digester实例，并设置相关规则属性，Digester用来解析xml，采用的是SAX方式。
    Digester digester = createStartDigester();

    InputSource inputSource = null;
    InputStream inputStream = null;
    File file = null;
    try {
        try {
            //配置文件，默认是conf/server.xml
            file = configFile();
            inputStream = new FileInputStream(file);
            inputSource = new InputSource(file.toURI().toURL().toString());
        } catch (Exception e) {。。。}
        if (inputStream == null) {
            try {
                inputStream = getClass().getClassLoader()
                    .getResourceAsStream(getConfigFile());
                inputSource = new InputSource
                    (getClass().getClassLoader()
                     .getResource(getConfigFile()).toString());
            } catch (Exception e) {。。。}
        }

        //上面没有找到配置文件，就找server-embed.xml
        if( inputStream==null ) {
            try {
                inputStream = getClass().getClassLoader()
                        .getResourceAsStream("server-embed.xml");
                inputSource = new InputSource
                (getClass().getClassLoader()
                        .getResource("server-embed.xml").toString());
            } catch (Exception e) {。。。}
        }
        if (inputStream == null || inputSource == null) {
            。。。
            return;
        }
        //解析配置文件
        try {
            inputSource.setByteStream(inputStream);
            digester.push(this);
            digester.parse(inputSource);
        } catch (SAXParseException spe) {。。。}
    } finally {
        if (inputStream != null) {
            try {
                inputStream.close();
            } catch (IOException e) {。。。}
        }
    }
    //解析配置文件中会实例化server，是一个StandardServer
    //设置server的catalina
    getServer().setCatalina(this);

    //重定向流
    //使用自定义的PrintStream代替System.out和System.err
    initStreams();

    //初始化新的server
    try {
        //对Server进行初始化
        getServer().init();
    } catch (LifecycleException e) {。。。}
    //启动结束时间
    long t2 = System.nanoTime();
}
```
load方法做的事情如下：

- 初始化目录，还有embedded的目录处理。
- 初始化命名空间。
- 创建并配置一个Digester实例，并设置相关规则属性，Digester用来解析xml，采用的是SAX方式。
- 读取解析配置文件。
- 调用解析配置文件时候已经初始化过的StandardServer的init方法，对Server进行初始化。

前面的步骤先不解析，直接看Server的init方法，刚方法实现在LifecycleBase类中。

### 初始化Server
LifecycleBase中的init方法如下：

```
public final synchronized void init() throws LifecycleException {
    //状态不为new，抛异常
    if (!state.equals(LifecycleState.NEW)) {
        invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
    }

    try {
        //设置状态为INITIALIZING
        setStateInternal(LifecycleState.INITIALIZING, null, false);
        //初始化
        initInternal();
        //设置状态为INITIALIZED
        setStateInternal(LifecycleState.INITIALIZED, null, false);
    } catch (Throwable t) {。。。}
}
```
继续往下看initInternal方法，在子类中实现，这里是在StandardServer中，对于下面的Engine，Host，Context的初始化也是这种步骤：

```
protected void initInternal() throws LifecycleException {
    //调用父类方法，设置oname
    super.initInternal();

    //？？？
    onameStringCache = register(new StringCache(), "type=StringCache");

    // 注册MBeanFactory
    MBeanFactory factory = new MBeanFactory();
    factory.setContainer(this);
    onameMBeanFactory = register(factory, "type=MBeanFactory");

    //注册naming资源
    globalNamingResources.init();

    // Populate the extension validator with JARs from common and shared
    // class loaders
    //使用从common和shared类加载器加载的类来填充扩展验证器
    if (getCatalina() != null) {
        ClassLoader cl = getCatalina().getParentClassLoader();
        //遍历类加载器，对于SystemClassLoader不处理
        while (cl != null && cl != ClassLoader.getSystemClassLoader()) {
            //URLClassLoader
            if (cl instanceof URLClassLoader) {
                URL[] urls = ((URLClassLoader) cl).getURLs();
                for (URL url : urls) {
                    if (url.getProtocol().equals("file")) {
                        try {
                            File f = new File (url.toURI());
                            if (f.isFile() &&
                                    f.getName().endsWith(".jar")) {
                                ExtensionValidator.addSystemResource(f);
                            }
                        } catch (URISyntaxException e) {
                            // Ignore
                        } catch (IOException e) {
                            // Ignore
                        }
                    }
                }
            }
            cl = cl.getParent();
        }
    }
    //调用各个service的init方法，一个Server下面可能有多个Service
    for (int i = 0; i < services.length; i++) {
        services[i].init();
    }
}
```

### 初始化Service
也是先调用LifecycleBase中的init方法，然后initInternal方法由子类实现，这里是StandardService：

```
protected void initInternal() throws LifecycleException {

    super.initInternal();
    //调用Container的init方法，这里是StandardEngine
    if (container != null) {
        container.init();
    }

    //实例化线程池
    for (Executor executor : findExecutors()) {
        if (executor instanceof LifecycleMBeanBase) {
            ((LifecycleMBeanBase) executor).setDomain(getDomain());
        }
        executor.init();
    }

    //实例化Connector，可能有多个Connector
    synchronized (connectorsLock) {
        for (Connector connector : connectors) {
            try {
                //调用Connector的init方法
                connector.init();
            } catch (Exception e) {...}
        }
    }
}
```
首先实例化Container，然后实例化Connector。

### 初始化Container
这里的Container是StandardEngine，看下init方法：

```
protected void initInternal() throws LifecycleException {
    //启动之前确保Realm存在
    getRealm();
    super.initInternal();
}
```
### 初始化Connector
Connector用来处理和客户端的通信相关内容。

Connector初始化，也是调用Connector的initInternal方法：

```
protected void initInternal() throws LifecycleException {

    super.initInternal();

    // CoyoteAdapter，用来处理请求
    adapter = new CoyoteAdapter(this);
    //protocolHandler对于不同请求会有不同的Handler
    //我们这里重点看http请求的处理，即是Http11Protocol
    protocolHandler.setAdapter(adapter);

    // Make sure parseBodyMethodsSet has a default
    if( null == parseBodyMethodsSet ) {
        setParseBodyMethods(getParseBodyMethods());
    }

    try {
        //处理器初始化
        protocolHandler.init();
    } catch (Exception e) {。。。}

    // 初始化 mapper listener
    mapperListener.init();
}
```

初始化的时候先初始化Service，然后是Container，然后是Connector。往下还有对protocolHandler的初始化和mapperListener的初始化。

### 初始化protocolHandler
这里主要看下对于http请求的处理，对应的Handler为Http11Protocol。初始化方法为init，实现在Http11Protocol的父类AbstractHttp11JsseProtocol中：

```
public void init() throws Exception {
    // SSL implementation needs to be in place before end point is
    // initialized
    sslImplementation = SSLImplementation.getInstance(sslImplementationName);
    super.init();
}
```
调用父类的init方法：

```
public void init() throws Exception {
    if (oname == null) {
        // Component not pre-registered so register it
        oname = createObjectName();
        if (oname != null) {
            Registry.getRegistry(null, null).registerComponent(this, oname,
                null);
        }
    }

    if (this.domain != null) {
        try {
            tpOname = new ObjectName(domain + ":" +
                    "type=ThreadPool,name=" + getName());
            Registry.getRegistry(null, null).registerComponent(endpoint,
                    tpOname, null);
        } catch (Exception e) {。。。}
        rgOname=new ObjectName(domain +
                ":type=GlobalRequestProcessor,name=" + getName());
        Registry.getRegistry(null, null).registerComponent(
                getHandler().getGlobal(), rgOname, null );
    }

    String endpointName = getName();
    endpoint.setName(endpointName.substring(1, endpointName.length()-1));

    try {
        //endpoint的初始化
        endpoint.init();
    } catch (Exception ex) {。。。}
}
```

关于oname和domain先不讲解，主要看初始化endpoint。

#### 初始化endpoint
endpoint提供底层的网络io实现。init方法实现在AbstractEndpoint中：

```
public final void init() throws Exception {
    testServerCipherSuitesOrderSupport();
    if (bindOnInit) {
        //绑定方法，该对底层的Socket之类的进行处理了。
        bind();
        bindState = BindState.BOUND_ON_INIT;
    }
}
```
bind方法的实现在子类中，这里使用的是JioEndpoint：

```
public void bind() throws Exception {

    //初始化acceptor的默认线程总数
    if (acceptorThreadCount == 0) {
        acceptorThreadCount = 1;
    }
    //初始化最大连接数
    if (getMaxConnections() == 0) {
        setMaxConnections(getMaxThreadsInternal());
    }

    if (serverSocketFactory == null) {
        //https的处理
        if (isSSLEnabled()) {
            serverSocketFactory =
                handler.getSslImplementation().getServerSocketFactory(this);
        } else {
            //处理http请求的serversocket工厂
            serverSocketFactory = new DefaultServerSocketFactory(this);
        }
    }

    if (serverSocket == null) {
        //创建serverSocket
        try {
            if (getAddress() == null) {
                serverSocket = serverSocketFactory.createSocket(getPort(),
                        getBacklog());
            } else {
                serverSocket = serverSocketFactory.createSocket(getPort(),
                        getBacklog(), getAddress());
            }
        } catch (BindException orig) {。。。}
    }

}
```
创建ServerSocket代码如下：

```
public ServerSocket createSocket (int port, int backlog,
        InetAddress ifAddress) throws IOException {
    return new ServerSocket (port, backlog, ifAddress);
}
```
创建ServerSocket，根据端口号和地址直接返回一个ServerSocket实例。到这里对于Handler的处理已经完成。接下来初始化mapperListener。
### 初始化mapperListener
MapperListener用来建立配置的URL映射的管理，即Host，Context，Wrapper（Servlet）和URL的映射关系。

init方法在LifecycleMBeanBase的initInternal方法中实现，没有做啥事。

初始化完成之后，就开始调用Catalina的start方法进行启动了。

## 初始化过程总结
整个初始化过程层次还是很清晰的：

- 首先初始化Server
- 然后初始化Service
- 接着初始化Container，Executor，Connector
- 初始化Connector的时候，需要初始化protocolHandler和mapperListener
- 初始化protocolHandler，这里需要初始化endpoint，初始化endpoint就是返回一个ServerSocket实例

## start方法启动Catalina

Catalina的start方法代码如下：

```
public void start() {
    //没有Server实例，就走一遍load
    if (getServer() == null) {
        load();
    }
    //还是没有Server实例，有错误！
    if (getServer() == null) {
        log.fatal("Cannot start server. Server instance is not configured.");
        return;
    }

    long t1 = System.nanoTime();

    //启动新的Server
    try {
        //这里Server是StandardServer
        getServer().start();
    } catch (LifecycleException e) {
        try {
            //异常，需要销毁
            getServer().destroy();
        } catch (LifecycleException e1) {。。。}
        return;
    }

    long t2 = System.nanoTime();

    // 注册关闭钩子
    if (useShutdownHook) {
        if (shutdownHook == null) {
            shutdownHook = new CatalinaShutdownHook();
        }
        Runtime.getRuntime().addShutdownHook(shutdownHook);
        LogManager logManager = LogManager.getLogManager();
        if (logManager instanceof ClassLoaderLogManager) {
            ((ClassLoaderLogManager) logManager).setUseShutdownHook(
                    false);
        }
    }
    //如果是await状态
    if (await) {
        await();
        stop();
    }
}
```

这里主要的步骤是调用Server的start方法，启动服务。这里Server是StandardServer。

### 启动Server
StandardServer的start方法，start方法在LifecycleBase中：

```
public final synchronized void start() throws LifecycleException {

    if (LifecycleState.STARTING_PREP.equals(state) || LifecycleState.STARTING.equals(state) ||
            LifecycleState.STARTED.equals(state)) {
        return;
    }
    //init
    if (state.equals(LifecycleState.NEW)) {
        init();
    } else if (state.equals(LifecycleState.FAILED)) {
        //stop
        stop();
    } else if (!state.equals(LifecycleState.INITIALIZED) &&
            !state.equals(LifecycleState.STOPPED)) {
        invalidTransition(Lifecycle.BEFORE_START_EVENT);
    }

    try {
        setStateInternal(LifecycleState.STARTING_PREP, null, false);
        startInternal();
        if (state.equals(LifecycleState.FAILED)) {
            stop();
        } else if (!state.equals(LifecycleState.STARTING)) {
            invalidTransition(Lifecycle.AFTER_START_EVENT);
        } else {
            setStateInternal(LifecycleState.STARTED, null, false);
        }
    } catch (Throwable t) {。。。}
}
```
上面也是调用startInternal()方法进行启动，在子类中实现，这里是在StandardServer中：

```
protected void startInternal() throws LifecycleException {
    //暂先不解释
    fireLifecycleEvent(CONFIGURE_START_EVENT, null);
    setState(LifecycleState.STARTING);
    //暂先不解释
    globalNamingResources.start();

    //启动Service
    synchronized (servicesLock) {
        for (int i = 0; i < services.length; i++) {
            services[i].start();
        }
    }
}
```
这里调用Service方法的start启动Service。

### 启动Service
StandardService的start方法，具体的实现是在StandardService的startInternal方法中：

```
protected void startInternal() throws LifecycleException {

    setState(LifecycleState.STARTING);

    //启动Container
    if (container != null) {
        synchronized (container) {
            container.start();
        }
    }
    //启动线程池
    synchronized (executors) {
        for (Executor executor: executors) {
            executor.start();
        }
    }

    //启动Connector
    synchronized (connectorsLock) {
        for (Connector connector: connectors) {
            try {
                if (connector.getState() != LifecycleState.FAILED) {
                    connector.start();
                }
            } catch (Exception e) {。。。}
        }
    }
}
```

### 启动Container
StandardEngine的start方法，也是调用startInternal方法，实现在StandardEngine中：

 ```
 protected synchronized void startInternal() throws LifecycleException {
     //标准容器启动，在父类中实现
     super.startInternal();
 }
 ```
  继续看ContainerBase的startInternal方法：

  ```
  //暂未解析
  protected synchronized void startInternal() throws LifecycleException {

      // Start our subordinate components, if any
      if ((loader != null) && (loader instanceof Lifecycle))
          ((Lifecycle) loader).start();
      logger = null;
      getLogger();
      if ((manager != null) && (manager instanceof Lifecycle))
          ((Lifecycle) manager).start();
      if ((cluster != null) && (cluster instanceof Lifecycle))
          ((Lifecycle) cluster).start();
      Realm realm = getRealmInternal();
      if ((realm != null) && (realm instanceof Lifecycle))
          ((Lifecycle) realm).start();
      if ((resources != null) && (resources instanceof Lifecycle))
          ((Lifecycle) resources).start();

      //启动子容器
      Container children[] = findChildren();
      List<Future<Void>> results = new ArrayList<Future<Void>>();
      for (int i = 0; i < children.length; i++) {
          results.add(startStopExecutor.submit(new StartChild(children[i])));
      }

      boolean fail = false;
      for (Future<Void> result : results) {
          try {
              result.get();
          } catch (Exception e) {
              log.error(sm.getString("containerBase.threadedStartFailed"), e);
              fail = true;
          }

      }
      if (fail) {
          throw new LifecycleException(
                  sm.getString("containerBase.threadedStartFailed"));
      }

      // Start the Valves in our pipeline (including the basic), if any
      if (pipeline instanceof Lifecycle)
          ((Lifecycle) pipeline).start();


      setState(LifecycleState.STARTING);

      // Start our thread
      threadStart();

  }
  ```
### 启动Connector
Connector的start方法，Connector的startInternal方法：

```
protected void startInternal() throws LifecycleException {
    setState(LifecycleState.STARTING);

    try {
        protocolHandler.start();
    } catch (Exception e) {。。。}

    mapperListener.start();
}
```
Connector的启动跟初始化差不多，也是先启动protocolHandler，然后启动mapperListener。

### 启动protocolHandler
代码实现在AbstractProtocol中：

```
public void start() throws Exception {
    try {
        endpoint.start();
    } catch (Exception ex) {。。。}
}
```
这里启动endpoint，代码实现在AbstractEndpoint中：

```
public final void start() throws Exception {
    //没有绑定，先绑定
    if (bindState == BindState.UNBOUND) {
        bind();
        bindState = BindState.BOUND_ON_START;
    }
    //启动
    startInternal();
}
```
startInternal方法用来启动endpoint，我们这里还是以JIoEndpoint来分析，代码如下：

```
public void startInternal() throws Exception {

    if (!running) {
        running = true;
        paused = false;

        //创建工作者线程
        if (getExecutor() == null) {
            createExecutor();
        }
        //实例化连接数栅栏
        initializeConnectionLatch();
        //启动Acceptor线程
        startAcceptorThreads();

        //启动异步超时线程
        Thread timeoutThread = new Thread(new AsyncTimeout(),
                getName() + "-AsyncTimeout");
        timeoutThread.setPriority(threadPriority);
        timeoutThread.setDaemon(true);
        timeoutThread.start();
    }
}
```
### 启动mapperListener
MapperListener的startInternal方法：

```
public void startInternal() throws LifecycleException {

    setState(LifecycleState.STARTING);

    findDefaultHost();

    Engine engine = (Engine) connector.getService().getContainer();
    addListeners(engine);

    Container[] conHosts = engine.findChildren();
    for (Container conHost : conHosts) {
        Host host = (Host) conHost;
        if (!LifecycleState.NEW.equals(host.getState())) {
            // Registering the host will register the context and wrappers
            registerHost(host);
        }
    }
}
```
到这里就启动完成，tomcat启动完毕，有关启动的还没有详细解释，待续～～～
