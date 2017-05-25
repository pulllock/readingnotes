tomcat在启动的时候，会调用Catalina的load的方法启动一个新的Server实例，在这里会有关于Digester的使用，以及对server.xml的解析过程。

load方法的代码如下：

```
public void load() {

    ...

    // Create and execute our Digester
    Digester digester = createStartDigester();

    ...

        try {
            inputSource.setByteStream(inputStream);
            digester.push(this);
            digester.parse(inputSource);
        }
        ...

}
```
代码做了精简，只保留了Digester的最重要部分。首先是创建并配置要用来启动的Digester实例，然后获取到server.xml文件的输入流之后，使用digester进行解析。

# Digester介绍
在进行具体步骤的解析之前，首先看一下Digester的简单介绍，Digester用来处理xml，是对SAX的包装，所以也是基于文件流来解析xml的。Digester使用的步骤也很简单：

- 创建一个Digester实例
- 设置相关属性
- 设置具体的规则
- 调用parse方法进行解析

# createStartDigester
此方法用来创建和配置Digester，对应着上面的前三个步骤，具体代码如下：

```
protected Digester createStartDigester() {
    long t1=System.currentTimeMillis();
    //创建一个digester实例
    Digester digester = new Digester();
    //是否需要验证xml文档的合法性，false表示不需要进行DTD规则校验
    digester.setValidating(false);
    //是否需要进行节点设置规则校验
    digester.setRulesValidation(true);
    //将xml节点中的className作为假属性，不用调用默认的setter方法
    //在解析时，调用相应对象的setter方法来设置属性值，setter的参数就是节点属性，
    //而有className的话，则直接使用className来直接实例化对象
    HashMap<Class<?>, List<String>> fakeAttributes = new HashMap<Class<?>, List<String>>();
    ArrayList<String> attrs = new ArrayList<String>();
    attrs.add("className");
    fakeAttributes.put(Object.class, attrs);
    digester.setFakeAttributes(fakeAttributes);
    digester.setUseContextClassLoader(true);

    //下面添加各种规则
    //遇到xml中Server节点，就创建一个StandardServer对象
    digester.addObjectCreate("Server", "org.apache.catalina.core.StandardServer",  "className");
    //根据Server节点中的属性信息，调用属性的setter方法，比如说server节点中会有port=“8080”属性，则会调用setPort方法
    digester.addSetProperties("Server");
    //在上面的load方法中有个digester.push(this)，this对象就是栈顶了
    //这里将Server节点对应的对象作为参数，调用this对象，也就是Catalina对象的setServer方法
    digester.addSetNext("Server", "setServer", "org.apache.catalina.Server");
    //Server节点下的GlobalNamingResources节点，创建一个NamingResource对象
    digester.addObjectCreate("Server/GlobalNamingResources", "org.apache.catalina.deploy.NamingResources");
    digester.addSetProperties("Server/GlobalNamingResources");
    digester.addSetNext("Server/GlobalNamingResources", "setGlobalNamingResources", "org.apache.catalina.deploy.NamingResources");
    //Server下的Listener节点
    digester.addObjectCreate("Server/Listener",  null, // MUST be specified in the element "className");
    digester.addSetProperties("Server/Listener");
    digester.addSetNext("Server/Listener", "addLifecycleListener", "org.apache.catalina.LifecycleListener");
    //Server下的Service节点
    digester.addObjectCreate("Server/Service",  "org.apache.catalina.core.StandardService", "className");
    digester.addSetProperties("Server/Service");
    digester.addSetNext("Server/Service", "addService", "org.apache.catalina.Service");
    //Service节点下的Listener节点
    digester.addObjectCreate("Server/Service/Listener", null,  "className");
    digester.addSetProperties("Server/Service/Listener");
    digester.addSetNext("Server/Service/Listener", "addLifecycleListener", "org.apache.catalina.LifecycleListener");

    //Executor节点
    digester.addObjectCreate("Server/Service/Executor",  "org.apache.catalina.core.StandardThreadExecutor",  "className");
    digester.addSetProperties("Server/Service/Executor");

    digester.addSetNext("Server/Service/Executor", "addExecutor", "org.apache.catalina.Executor");

    //给Connector添加规则，就是当遇到Connector的时候，会调用ConnectorCreateRule里面定义的规则
    //跟上面的作用是一样的，只不过该节点的规则比较多，就创建一个规则类
    digester.addRule("Server/Service/Connector", new ConnectorCreateRule());
    digester.addRule("Server/Service/Connector",  new SetAllPropertiesRule(new String[]{"executor"}));
    digester.addSetNext("Server/Service/Connector", "addConnector", "org.apache.catalina.connector.Connector");


    digester.addObjectCreate("Server/Service/Connector/Listener", null,  "className");
    digester.addSetProperties("Server/Service/Connector/Listener");
    digester.addSetNext("Server/Service/Connector/Listener", "addLifecycleListener", "org.apache.catalina.LifecycleListener");

    //给嵌入元素添加RuleSet自定义规则
    //每个rule规则，都会有tomcat对自身业务逻辑的判断和处理
    digester.addRuleSet(new NamingRuleSet("Server/GlobalNamingResources/"));
    digester.addRuleSet(new EngineRuleSet("Server/Service/"));
    digester.addRuleSet(new HostRuleSet("Server/Service/Engine/"));
    digester.addRuleSet(new ContextRuleSet("Server/Service/Engine/Host/"));
    addClusterRuleSet(digester, "Server/Service/Engine/Host/Cluster/");
    digester.addRuleSet(new NamingRuleSet("Server/Service/Engine/Host/Context/"));

    // When the 'engine' is found, set the parentClassLoader.
    digester.addRule("Server/Service/Engine", new SetParentClassLoaderRule(parentClassLoader));
    addClusterRuleSet(digester, "Server/Service/Engine/Cluster/");

    return (digester);

}
```
上面创建完Digester对象，并设置了属性和各种规则之后，接下来主要的就是解析工作。

# parse解析
parse方法代码如下：

```
public Object parse(InputSource input) throws IOException, SAXException {
    //解析前的配置，默认什么也没做，需要子类去实现
    configure();
    //获取解析器去解析
    getXMLReader().parse(input);
    return (root);

}
```
解析完成之后，Server,Service等等在server.xml中存在的节点，就会有对应的对象存在，后续就可以使用这些对象进行初始化和启动了。
