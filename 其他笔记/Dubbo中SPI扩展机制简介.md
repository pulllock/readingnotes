前面我们了解过了Java的SPI扩展机制，对于Java扩展机制的原理以及优缺点也有了大概的了解，这里继续深入一下Dubbo的扩展点加载机制。

# Dubbo扩展点加载的功能
Dubbo的扩展点加载机制类似于Java的SPI，我们知道Java的SPI在使用的时候，只能通过遍历来进行实现的查找和实例化，有可能会一次性把所有的实现都实例化，这样会造成有些不使用的扩展实现也会被实例化，这就会造成一定的资源浪费。有关Dubbo的改进，参照文档上的说明：

- JDK标准的SPI会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
- 如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK标准的ScriptEngine，通过getName();获取脚本类型的名称，但如果RubyScriptEngine因为所依赖的jruby.jar不存在，导致RubyScriptEngine类加载失败，这个失败原因被吃掉了，和ruby对应不起来，当用户执行ruby脚本时，会报不支持ruby，而不是真正失败的原因。
- 增加了对扩展点IoC和AOP的支持，一个扩展点可以直接setter注入其它扩展点。

关于第一点，通过和Java的SPI对比，就能明白；第二点还未做测试，不太清楚其中的缘由；第三点对于IOC和AOP的支持下面简单介绍下。

## 扩展点自动装配功能（IOC）
就是当加载一个扩展点时，会自动的注入这个扩展点所依赖的**其他扩展点**，如果描述不清楚的话，可以看下下面的例子：

```
接口A，实现类A1，A2
接口B，实现类B1，B2
```
其中实现类A1含有setB()方法，当通过扩展机制加载A的实现的时候，会自动的注入一个B的实现类，但是，此时不是注入B1，也不是注入B2，而是注入一个自适应的B的实现类：`B$Adpative`，该实现类是动态生成的，能够根据参数的不同，自动选择B1或者B2来进行调用。

## 扩展点自适应
上面我们说，在自动装配的时候，并不是注入一个真正的实现，而是注入一个自适应的扩展点实现，其实就是动态的生成的代码，也就是手动拼装的代码，这段代码里会根据SPI上配置的信息来加入对于具体实现的选择功能。生成的代码类似于下面的，代码做了一下精简，把包都去掉了：

```
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class Protocol$Adpative implements Protocol {
  public Invoker refer(Class arg0, URL arg1) throws Class {
    if (arg1 == null) throw new IllegalArgumentException("url == null");

    URL url = arg1;
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );

    if(extName == null) throw new IllegalStateException("Fail to get extension(Protocol) name from url(" + url.toString() + ") use keys([protocol])");
    
    Protocol extension = (Protocol)ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName);
    
    return extension.refer(arg0, arg1);
  }
  
  public Exporter export(Invoker arg0) throws Invoker {
    if (arg0 == null) throw new IllegalArgumentException("Invoker argument == null");
    
    if (arg0.getUrl() == null) throw new IllegalArgumentException("Invoker argument getUrl() == null");URL url = arg0.getUrl();
    //这里会根据url中的信息获取具体的实现类名
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
    
    if(extName == null) throw new IllegalStateException("Fail to get extension(Protocol) name from url(" + url.toString() + ") use keys([protocol])");
    //根据上面的实现类名，会在运行时，通过Dubbo的扩展机制加载具体实现类
    Protocol extension = (Protocol)ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName);
    
    return extension.export(arg0);
  }
  
  public void destroy() {
  	throw new UnsupportedOperationException("method public abstract void Protocol.destroy() of interface Protocol is not adaptive method!");
  }
  
  public int getDefaultPort() {
  	throw new UnsupportedOperationException("method public abstract int Protocol.getDefaultPort() of interface Protocol is not adaptive method!");
  }
}
```
使用这种方式的原因也很容易能想到，在我们加载扩展点实现的时候，并没有调用实现的具体逻辑，那我们注入一个扩展点，也就不知道这个扩展点的实现具体是什么，所以要注入一个自适应的实现。等到运行时候，才根据自适应实现，来调用真正实现。

## 扩展点自动包装功能（AOP）

先看下下面的示例，假如接口A还有另外一个实现者：AWrapper1：

```
class AWrapper1 implements A{
	private A a;
    AWrapper1(A a){
      this.a = a;
    }
    
}
```
AWrapper1相当于A的包装类，类似于AOP的功能，AWrapper1增加了A的功能。当我们获取接口A的实现类的时候，得到的就是包装过的类。

# Dubbo扩展点加载的实现
首先还是定义接口，然后是接口的具体实现类，配置文件类似于Java的SPI配置文件，Dubbo的配置文件放在`META-INF/dubbo/`目录下，配置文件名为接口的全限定名，配置文件内容是`配置名=扩展实现类的全限定名`，加载实现类的功能是通过ExtensionLoader来实现，类似于Java中的ServiceLoader的作用。

另外，扩展点使用单一实例加载，需要确保线程安全性。

# Dubbo扩展点加载的一些定义
- `@SPI`注解，被此注解标记的接口，就表示是一个可扩展的接口。
- `@Adaptive`注解，有两种注解方式：一种是注解在类上，一种是注解在方法上。

	- 注解在类上，而且是注解在实现类上，目前dubbo只有AdaptiveCompiler和AdaptiveExtensionFactory类上标注了此注解，这是些特殊的类，ExtensionLoader需要依赖他们工作，所以得使用此方式。
	- 注解在方法上，注解在接口的方法上，除了上面两个类之外，所有的都是注解在方法上。ExtensionLoader根据接口定义动态的生成适配器代码，并实例化这个生成的动态类。被Adaptive注解的方法会生成具体的方法实现。没有注解的方法生成的实现都是抛不支持的操作异常UnsupportedOperationException。被注解的方法在生成的动态类中，会根据url里的参数信息，来决定实际调用哪个扩展。

	比如说这段代码：

    ```
    private static final Protocol refprotocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
    ```

	当上面代码执行的时候，我们其实还不知道要真正使用的Protocol是什么，可能是具体的实现DubboProtocol，也可能是其他的具体实现的Protocol，那么这时候refprotocol到底是什么呢？refprotocol其实是在调用getAdaptiveExtension()方法时候，自动生成的一个类，代码如下：

    ```
    import com.alibaba.dubbo.common.extension.ExtensionLoader;
    public class Protocol$Adpative implements Protocol {
      public Invoker refer(Class arg0, URL arg1) throws Class {
        if (arg1 == null) throw new IllegalArgumentException("url == null");

        URL url = arg1;
        String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );

        if(extName == null) throw new IllegalStateException("Fail to get extension(Protocol) name from url(" + url.toString() + ") use keys([protocol])");

        Protocol extension = (Protocol)ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName);

        return extension.refer(arg0, arg1);
      }

      public Exporter export(Invoker arg0) throws Invoker {
        if (arg0 == null) throw new IllegalArgumentException("Invoker argument == null");

        if (arg0.getUrl() == null) throw new IllegalArgumentException("Invoker argument getUrl() == null");URL url = arg0.getUrl();

        String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );

        if(extName == null) throw new IllegalStateException("Fail to get extension(Protocol) name from url(" + url.toString() + ") use keys([protocol])");

        Protocol extension = (Protocol)ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName);

        return extension.export(arg0);
      }

      public void destroy() {
        throw new UnsupportedOperationException("method public abstract void Protocol.destroy() of interface Protocol is not adaptive method!");
      }

      public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int Protocol.getDefaultPort() of interface Protocol is not adaptive method!");
      }
    }
    ```
	可以看到被@Adaptive注解的方法都生成了具体的实现，并且实现逻辑都相同。而没有被注解的方法直接抛出不支持操作的异常。

	当我们使用refprotocol调用方法的时候，其实是调用生成的类`Protocol$Adpative`中的方法，这里面的方法根据url中的参数配置来找到具体的实现类，找具体实现类的方式还是通过dubbo的扩展机制。比如url中可能会有protocol=dubbo，此时就可以根据这个dubbo来确定我们要找的类是DubboProtocol。可以查看下生成的代码中`getExtension(extName)`这里是根据具体的名字去查找实现类。

- `@Activate`注解，此注解需要注解在类上或者方法上，并注明被激活的条件，以及所有的被激活实现类中的排序信息。

- ExtensionLoader，是dubbo的SPI机制的查找服务实现的工具类，类似与Java的ServiceLoader，可做类比。dubbo约定扩展点配置文件放在classpath下的`/META-INF/dubbo，/META-INF/dubbo/internal，/META-INF/services`目录下，配置文件名为接口的全限定名，配置文件内容为`配置名=扩展实现类的全限定名`。

# Dubbo扩展点加载的源码解析

重点解析下ExtensionLoader这个类。Dubbo的扩展点使用单一实例去加载，缓存在ExtensionLoader中。每一个ExtensionLoader实例仅负责加载特定SPI扩展的实现，想要获得某个扩展的实现，首先要获得该扩展对应的ExtensionLoader实例。

以Protocol为例进行分析扩展点的加载：

```
//这样使用，先获取ExtensionLoader实例，然后加载自适应的Protocol扩展点
Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
//使用
protocol.refer(Class<T> type, URL url))；
```
可以看到，使用扩展点加载的步骤大概有三步：

1. 获取ExtensionLoader实例。
2. 获取自适应实现。
3. 使用获取到的实现。

下面我们就以这三步作为分界，来深入源码的解析。

## 获取ExtensionLoader实例
第一步，getExtensionLoader(Protocol.class)，根据要加载的接口Protocol，创建出一个ExtensionLoader实例，加载完的实例会被缓存起来，下次再加载Protocol的ExtensionLoader的时候，会使用已经缓存的这个，不会再新建一个实例：

```
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    //扩展点类型不能为空
    if (type == null)
        throw new IllegalArgumentException();
    //扩展点类型只能是接口类型的
    if(!type.isInterface()) {
        throw new IllegalArgumentException();
    }
    //没有添加@SPI注解，只有注解了@SPI的才会解析
    if(!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException();
    }
    //先从缓存中获取指定类型的ExtensionLoader
    //EXTENSION_LOADERS是一个ConcurrentHashMap，缓存了所有已经加载的ExtensionLoader的实例
    //比如这里加载Protocol.class，就以Protocol.class作为key，以新创建的ExtensionLoader作为value
    //每一个要加载的扩展点只会对应一个ExtensionLoader实例，也就是只会存在一个Protocol.class在缓存中
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    //缓存中不存在
    if (loader == null) {
        //创建一个新的ExtensionLoader实例，放到缓存中去
        //对于每一个扩展，dubbo中只有一个对应的ExtensionLoader实例
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```
上面代码返回一个ExtensionLoader实例，`getExtensionLoader(Protocol.class)`这一步没有进行任何的加载工作，只是获得了一个ExtensionLoader的实例。

### ExtensionLoader的构造方法
上面获取的是一个ExtensionLoader实例，接着看下构造实例的时候到底做了什么，我们发现在ExtensionLoader中只有一个私有的构造方法：

```
private ExtensionLoader(Class<?> type) {
	//接口类型
    this.type = type;
    //对于扩展类型是ExtensionFactory的，设置为null
    //getAdaptiveExtension方法获取一个运行时自适应的扩展类型
    //每个Extension只能有一个@Adaptive类型的实现，如果么有，dubbo会自动生成一个类
    //objectFactory是一个ExtensionFactory类型的属性，主要用于加载扩展的实现
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```
不难理解，ExtensionFactory是

加载是在调用getAdaptiveExtension()方法中进行的：

```
getAdaptiveExtension()-->
                createAdaptiveExtension()-->
                                getAdaptiveExtensionClass()-->
                                                getExtensionClasses()-->
                                                                loadExtensionClasses()

```

## 获取自适应实现

先看下getAdaptiveExtension()方法，用来获取一个扩展的自适应实现类，最后返回的自适应实现类是一个类名为`Protocol$Adaptive`的类，并且这个类实现了Protocol接口：

```
public T getAdaptiveExtension() {
    //先从实例缓存中查找实例对象
    //private final Holder<Object> cachedAdaptiveInstance = new Holder<Object>();
    //在当前的ExtensionLoader中保存着一个Holder实例，用来缓存自适应实现类的实例
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {//缓存中不存在
        if(createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                //获取锁之后再检查一次缓存中是不是已经存在
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        //缓存中没有，就创建新的AdaptiveExtension实例
                        instance = createAdaptiveExtension();
                        //新实例加入缓存
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {createAdaptiveInstanceError = t; }
                }
            }
        }
    }

    return (T) instance;
}
```
### 创建自适应扩展

接着看下createAdaptiveExtension()方法，用来创建自适应扩展类的实例：

```
private T createAdaptiveExtension() {
    try {
        //先通过getAdaptiveExtensionClass获取AdaptiveExtensionClass
        //然后获取其实例
        //最后进行注入处理
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {}
}
```
### 获取自适应扩展类
接着查看getAdaptiveExtensionClass()方法，用来获取一个自适应扩展的Class，这个Class将会在下一步被实例化：

```
private Class<?> getAdaptiveExtensionClass() {
    //加载当前Extension的所有实现（这里举例是Protocol，只会加载Protocol的所有实现类），如果有@Adaptive类型的实现类，会赋值给cachedAdaptiveClass
    //目前只有AdaptiveExtensionFactory和AdaptiveCompiler两个实现类是被注解了@Adaptive
    getExtensionClasses();
    //加载完所有的实现之后，发现有cachedAdaptiveClass不为空
    //也就是说当前获取的自适应实现类是AdaptiveExtensionFactory或者是AdaptiveCompiler，就直接返回，这两个类是特殊用处的，不用代码生成，而是现成的代码
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    //没有找到Adaptive类型的实现，动态创建一个
    //比如Protocol的实现类，没有任何一个是用@Adaptive来直接的，只有Protocol接口的方法是有注解的
    //这时候就需要来动态的生成了，也就是生成Protocol$Adaptive
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```
#### 加载扩展类实现
先看下getExtensionClasses()这个方法，加载所有的扩展类的实现：

```
private Map<String, Class<?>> getExtensionClasses() {
    //从缓存中获取，cachedClasses也是一个Holder，Holder这里持有的是一个Map，key是扩展点实现名，value是扩展点实现类
    //这里会存放当前扩展点类型的所有的扩展点的实现类
    //这里以Protocol为例，就是会存放Protocol的所有实现类
    //比如key为dubbo，value为com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol
    //cachedClasses扩展点实现名称对应的实现类
    Map<String, Class<?>> classes = cachedClasses.get();
    //如果为null，说明没有被加载过，就会进行加载，而且加载就只会进行这一次
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                //如果没有加载过Extension的实现，进行扫描加载，完成后缓存起来
                //每个扩展点，其实现的加载只会这执行一次
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```

看下loadExtensionClasses()方法，这个方法中加载扩展点的实现类：

```
private Map<String, Class<?>> loadExtensionClasses() {
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if(defaultAnnotation != null) {
        //当前Extension的默认实现名字
        //比如说Protocol接口，注解是@SPI("dubbo")
        //这里dubbo就是默认的值
        String value = defaultAnnotation.value();
        //只能有一个默认的名字，如果多了，谁也不知道该用哪一个实现了。
        if(value != null && (value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if(names.length > 1) {
                throw new IllegalStateException();
            }
            //默认的名字保存起来
            if(names.length == 1) cachedDefaultName = names[0];
        }
    }

    //下面就开始从配置文件中加载扩展实现类
    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    //从META-INF/dubbo/internal目录下加载
    loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
    //从META-INF/dubbo/目录下加载
    loadFile(extensionClasses, DUBBO_DIRECTORY);
    //从META-INF/services/下加载
    loadFile(extensionClasses, SERVICES_DIRECTORY);
    return extensionClasses;
}
```
从各个位置的配置文件中加载实现类，对于Protocol来说加载的文件是以`com.alibaba.dubbo.rpc.Protocol`为名称的文件，文件的内容是（有好几个同名的配置文件，这里直接把内容全部写在了一起）：

```
registry=com.alibaba.dubbo.registry.integration.RegistryProtocol

filter=com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper
listener=com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper
mock=com.alibaba.dubbo.rpc.support.MockProtocol

dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol

hessian=com.alibaba.dubbo.rpc.protocol.hessian.HessianProtocol

com.alibaba.dubbo.rpc.protocol.http.HttpProtocol

injvm=com.alibaba.dubbo.rpc.protocol.injvm.InjvmProtocol

memcached=memcom.alibaba.dubbo.rpc.protocol.memcached.MemcachedProtocol

redis=com.alibaba.dubbo.rpc.protocol.redis.RedisProtocol

rmi=com.alibaba.dubbo.rpc.protocol.rmi.RmiProtocol

thrift=com.alibaba.dubbo.rpc.protocol.thrift.ThriftProtocol

com.alibaba.dubbo.rpc.protocol.webservice.WebServiceProtocol
```

看下loadFile()方法：

```
private void loadFile(Map<String, Class<?>> extensionClasses, String dir) {
    //配置文件的名称
    //这里type是扩展类，比如com.alibaba.dubbo.rpc.Protocol类
    String fileName = dir + type.getName();
    try {
        Enumeration<java.net.URL> urls;
        //获取类加载器
        ClassLoader classLoader = findClassLoader();
        //获取对应配置文件名的所有的文件
        if (classLoader != null) {
            urls = classLoader.getResources(fileName);
        } else {
            urls = ClassLoader.getSystemResources(fileName);
        }
        if (urls != null) {
            //遍历文件进行处理
            while (urls.hasMoreElements()) {
                //配置文件路径
                java.net.URL url = urls.nextElement();
                try {
                    BufferedReader reader = new BufferedReader(new InputStreamReader(url.openStream(), "utf-8"));
                    try {
                        String line = null;
                        //每次处理一行
                        while ((line = reader.readLine()) != null) {
                            //#号以后的为注释
                            final int ci = line.indexOf('#');
                            //注释去掉
                            if (ci >= 0) line = line.substring(0, ci);
                            line = line.trim();
                            if (line.length() > 0) {
                                try {
                                    String name = null;
                                    //=号之前的为扩展名字，后面的为扩展类实现的全限定名
                                    int i = line.indexOf('=');
                                    if (i > 0) {
                                        name = line.substring(0, i).trim();
                                        line = line.substring(i + 1).trim();
                                    }
                                    if (line.length() > 0) {
                                        //加载扩展类的实现
                                        Class<?> clazz = Class.forName(line, true, classLoader);
                                        //查看类型是否匹配
                                        //type是Protocol接口
                                        //clazz就是Protocol的各个实现类
                                        if (! type.isAssignableFrom(clazz)) {
                                            throw new IllegalStateException();
                                        }
                                        //如果实现类是@Adaptive类型的，会赋值给cachedAdaptiveClass，这个用来存放被@Adaptive注解的实现类
                                        if (clazz.isAnnotationPresent(Adaptive.class)) {
                                            if(cachedAdaptiveClass == null) {
                                                cachedAdaptiveClass = clazz;
                                            } else if (! cachedAdaptiveClass.equals(clazz)) {
                                                throw new IllegalStateException();
                                            }
                                        } else {//不是@Adaptice类型的类，就是没有注解@Adaptive的实现类
                                            try {//判断是否是wrapper类型
                                            	//如果得到的实现类的构造方法中的参数是扩展点类型的，就是一个Wrapper类
                                                //比如ProtocolFilterWrapper，实现了Protocol类，
                                                //而它的构造方法是这样public ProtocolFilterWrapper(Protocol protocol)
                                                //就说明这个类是一个包装类
                                                clazz.getConstructor(type);
                                                //cachedWrapperClasses用来存放当前扩展点实现类中的包装类
                                                Set<Class<?>> wrappers = cachedWrapperClasses;
                                                if (wrappers == null) {
                                                    cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                                                    wrappers = cachedWrapperClasses;
                                                }
                                                wrappers.add(clazz);
                                            } catch (NoSuchMethodException e) {
                                            //没有上面提到的构造器，则说明不是wrapper类型
                                            	//获取无参构造
                                                clazz.getConstructor();
                                                //没有名字，就是配置文件中没有xxx=xxxx.com.xxx这种
                                                if (name == null || name.length() == 0) {
                                                	//去找@Extension注解中配置的值
                                                    name = findAnnotationName(clazz);
                                                    //如果还没找到名字，从类名中获取
                                                    if (name == null || name.length() == 0) {
                                                    	//比如clazz是DubboProtocol，type是Protocol
                                                        //这里得到的name就是dubbo
                                                        if (clazz.getSimpleName().length() > type.getSimpleName().length()
                                                                && clazz.getSimpleName().endsWith(type.getSimpleName())) {
                                                            name = clazz.getSimpleName().substring(0, clazz.getSimpleName().length() - type.getSimpleName().length()).toLowerCase();
                                                        } else {
                                                            throw new IllegalStateException(");
                                                        }
                                                    }
                                                }
                                                //有可能配置了多个名字
                                                String[] names = NAME_SEPARATOR.split(name);
                                                if (names != null && names.length > 0) {
                                                    //是否是Active类型的类
                                                    Activate activate = clazz.getAnnotation(Activate.class);
                                                    if (activate != null) {
                                                    	//第一个名字作为键，放进cachedActivates这个map中缓存
                                                        cachedActivates.put(names[0], activate);
                                                    }
                                                    for (String n : names) {
                                                        if (! cachedNames.containsKey(clazz)) {
                                                            //放入Extension实现类与名称映射的缓存中去，每个class只对应第一个名称有效
                                                            cachedNames.put(clazz, n);
                                                        }
                                                        Class<?> c = extensionClasses.get(n);
                                                        if (c == null) {
                                                            //放入到extensionClasses缓存中去，多个name可能对应一份extensionClasses
                                                            extensionClasses.put(n, clazz);
                                                        } else if (c != clazz) {
                                                            throw new IllegalStateException();
                                                        }
                                                    }
                                                }
                                            }
                                        }
                                    }
                                } catch (Throwable t) { }
                            }
                        } // end of while read lines
                    } finally {
                        reader.close();
                    }
                } catch (Throwable t) { }
            } // end of while urls
        }
    } catch (Throwable t) { }
}
```
到这里加载当前Extension的所有实现就已经完成了，继续返回getAdaptiveExtensionClass中，在调用完getExtensionClasses()之后，会首先检查是不是已经有@Adaptive注解的类被解析并加入到缓存中了，如果有就直接返回，这里的cachedAdaptiveClass中现在只能是AdaptiveExtensionFactory或者AdaptiveCompiler中的一个，如果没有，说明是一个普通扩展点，就动态创建一个，比如会创建一个`Protocol$Adaptive`。

#### 创建自适应扩展类的代码
看下createAdaptiveExtensionClass()这个方法，用来动态的创建自适应扩展类：

```
private Class<?> createAdaptiveExtensionClass() {
	//组装自适应扩展点类的代码
    String code = createAdaptiveExtensionClassCode();
    //获取到应用的类加载器
    ClassLoader classLoader = findClassLoader();
    //获取编译器
    //dubbo默认使用javassist
    //这里还是使用扩展点机制来找具体的Compiler的实现
    //现在就知道cachedAdaptiveClass是啥意思了，如果没有AdaptiveExtensionFactory和AdaptiveCompiler这两个类，这里又要去走加载流程然后来生成扩展点类的代码，不就死循环了么。
    //这里解析Compiler的实现类的时候，会在getAdaptiveExtensionClass中直接返回
    //可以查看下AdaptiveCompiler这个类，如果我们没有指定，默认使用javassist
    //这里Compiler是JavassistCompiler实例
    com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    //将代码转换成Class
    return compiler.compile(code, classLoader);
}
```

接着看下createAdaptiveExtensionClassCode()方法，用来组装自适应扩展类的代码（拼写源码，代码比较长不在列出），这里列出生成的`Protocol$Adaptive`：

```
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {
  public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws java.lang.Class {
    if (arg1 == null) throw new IllegalArgumentException("url == null");

    com.alibaba.dubbo.common.URL url = arg1;
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );

    if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
    
    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
    
    return extension.refer(arg0, arg1);
  }
  
  public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.Invoker {
    if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
    
    if (arg0.getUrl() == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();
    
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
    
    if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
    
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
其他具体的扩展点的生成也类似。在生成完代码之后，是找到ClassLoader，然后获取到Compiler的自适应实现，这里得到的就是AdaptiveCompiler，最后调用`compiler.compile(code, classLoader);`来编译上面生成的类并返回，先进入AdaptiveCompiler的compile方法：

```
public Class<?> compile(String code, ClassLoader classLoader) {
    Compiler compiler;
    //得到一个ExtensionLoader
    ExtensionLoader<Compiler> loader = ExtensionLoader.getExtensionLoader(Compiler.class);
    //默认的Compiler名字
    String name = DEFAULT_COMPILER; // copy reference
    //有指定了Compiler名字，就使用指定的名字来找到Compiler实现类
    if (name != null && name.length() > 0) {
        compiler = loader.getExtension(name);
    } else {//没有指定Compiler名字，就查找默认的Compiler的实现类
        compiler = loader.getDefaultExtension();
    }
    //调用具体的实现类来进行编译
    return compiler.compile(code, classLoader);
}
```
#### 获取指定名字的扩展
先看下根据具体的名字来获取扩展的实现类`loader.getExtension(name);`，loader是`ExtensionLoader<Compiler> `类型的。这里就是比Java的SPI要方便的地方，Java的SPI只能通过遍历所有的实现类来查找，而dubbo能够指定一个名字查找。代码如下：

```
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    //如果name指定为true，则获取默认实现
    if ("true".equals(name)) {
    	//默认实现查找在下面解析
        return getDefaultExtension();
    }
    //先从缓存获取Holder，cachedInstance是一个ConcurrentHashMap，键是扩展的name，值是一个持有name对应的实现类实例的Holder。
    Holder<Object> holder = cachedInstances.get(name);
    //如果当前name对应的Holder不存在，就创建一个，添加进map中
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    //从Holder中获取保存的实例
    Object instance = holder.get();
    //不存在，就需要根据这个name找到实现类，实例化一个
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                //缓存不存在，创建实例
                instance = createExtension(name);
                //加入缓存
                holder.set(instance);
            }
        }
    }
    //存在，就直接返回
    return (T) instance;
}
```

创建扩展实例，`createExtension(name);`：

```
private T createExtension(String name) {
    //getExtensionClasses加载当前Extension的所有实现
    //上面已经解析过，返回的是一个Map，键是name，值是name对应的Class
    //根据name查找对应的Class
    Class<?> clazz = getExtensionClasses().get(name);
    //如果这时候class还不存在，说明在所有的配置文件中都没找到定义，抛异常
    if (clazz == null) {
        throw findException(name);
    }
    try {
        //从已创建实例缓存中获取
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        //不存在的话就创建一个新实例，加入到缓存中去
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        //属性注入
        injectExtension(instance);
        //Wrapper的包装
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && wrapperClasses.size() > 0) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) { }
}
```
有关属性注入和Wrapper的包装，下面再讲。到这里Compiler就能获得到一个指定name的具体实现类的实例了，然后就是调用实例的compile()方法对生成的代码进行编译。

#### 获取默认扩展实现
如果在AdaptiveCompiler中没有找到指定的名字，就会找默认的扩展实现`loader.getDefaultExtension();`：

```
public T getDefaultExtension() {
	//首先还是先去加载所有的扩展实现
    //加载的时候会设置默认的名字cachedDefaultName，这个名字是在@SPI中指定的，比如Compiler就指定了@SPI("javassist")，所以这里是javassist
    getExtensionClasses();
    if(null == cachedDefaultName || cachedDefaultName.length() == 0
            || "true".equals(cachedDefaultName)) {
        return null;
    }
    //根据javassist这个名字去查找扩展实现
    //具体的过程上面已经解析过了
    return getExtension(cachedDefaultName);
}
```
关于javassist编译Class的过程暂先不说明。我们接着流程看：

```
 private T createAdaptiveExtension() {
    try {
        //先通过getAdaptiveExtensionClass获取AdaptiveExtensionClass（在上面这一步已经解析了，获得到了一个自适应实现类的Class）
        //然后获取其实例，newInstance进行实例
        //最后进行注入处理injectExtension
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) { }
}
```

### 扩展点注入

接下来就是有关扩展点的注入的问题了，injectExtension，关于注入的解释查看最上面扩展点自动装配（IOC）的说明，injectExtension方法：

```
//这里的实例是Xxxx$Adaptive
private T injectExtension(T instance) {
    try {
    	//关于objectFactory的来路，先看下面的解析
        //这里的objectFactory是AdaptiveExtensionFactory
        if (objectFactory != null) {
        	//遍历扩展实现类实例的方法
            for (Method method : instance.getClass().getMethods()) {
                //只处理set方法
                //set开头，只有一个参数，public
                if (method.getName().startsWith("set")
                        && method.getParameterTypes().length == 1
                        && Modifier.isPublic(method.getModifiers())) {
                    //set方法参数类型
                    Class<?> pt = method.getParameterTypes()[0];
                    try {
                        //setter方法对应的属性名
                        String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                        //根据类型和名称信息从ExtensionFactory中获取
                        //比如在某个扩展实现类中会有setProtocol(Protocol protocol)这样的set方法
                        //这里pt就是Protocol，property就是protocol
                        //AdaptiveExtensionFactory就会根据这两个参数去查找对应的扩展实现类
                        //这里就会返回Protocol$Adaptive
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {//说明set方法的参数是扩展点类型，进行注入
                            //为set方法注入一个自适应的实现类
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) { }
                }
            }
        }
    } catch (Exception e) {}
    return instance;
}
```

有关AdaptiveExtensionFactory中获取Extension的过程，会首先在实例化的时候得到ExtensionFactory的具体实现类，然后遍历每个ExtensionFactory的实现类，分别在每个ExtensionFactory的实现类中获取Extension。

这里使用SpiExtensionFactory的获取扩展的方法为例，getExtension，也是先判断给定类是否是注解了@SPI的接口，然后根据类去获取ExtensionLoader，在使用得到的ExtensionLoader去加载自适应扩展。

#### objectFactory的来历
objectFactory的来路，在ExtensionLoader中有个私有构造器：

```
//当我们调用getExtensionLoader这个静态方法的时候，会触发ExtensionLoader类的实例化，会先初始化静态变量和静态块，然后是构造代码块，最后是构造器的初始化
private ExtensionLoader(Class<?> type) {
    this.type = type;
    //这里会获得一个AdaptiveExtensionFactory
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```

到此为止createAdaptiveExtension方法解析完成，接着返回上层getAdaptiveExtension()方法中，发现创建完自适应扩展实例之后，就会加入到cachedAdaptiveInstance缓存起来，然后就会返回给调用的地方一个`Xxx$Adaptive`实例。

走到这里，下面的代码就解析完了：

```
private static final Protocol refprotocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

## 得到扩展之后的使用
我们得到了一个`Protocol$Adaptive`实例，接着就是调用了，比如说我们要调用`refprotocol.refer(Class<T> type, URL url))`方法，由于这里refprotocol是一个`Protocol$Adaptive`实例，所以就先调用这个实例的refer方法，这里的实例的代码在最上面：

```
//这里为了好看，代码做了精简，包名都去掉了
public Invoker refer(Class arg0, URL arg1) throws Class {
    if (arg1 == null) throw new IllegalArgumentException();

    URL url = arg1;
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );

    if(extName == null) throw new IllegalStateException();
    
    Protocol extension = (Protocol)ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName);
    
    return extension.refer(arg0, arg1);
  }
```
可以看到这里首先根据url中的参数获取扩展名字，如果url中没有就使用默认的扩展名，然后根据扩展名去获取具体的实现。关于getExtension(String name)上面已经解析过一次，这里再次列出：

```
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    //如果name指定为true，则获取默认实现
    if ("true".equals(name)) {
    	//默认实现查找在下面解析
        return getDefaultExtension();
    }
    //先从缓存获取Holder，cachedInstance是一个ConcurrentHashMap，键是扩展的name，值是一个持有name对应的实现类实例的Holder。
    Holder<Object> holder = cachedInstances.get(name);
    //如果当前name对应的Holder不存在，就创建一个，添加进map中
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    //从Holder中获取保存的实例
    Object instance = holder.get();
    //不存在，就需要根据这个name找到实现类，实例化一个
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                //缓存不存在，创建实例
                instance = createExtension(name);
                //加入缓存
                holder.set(instance);
            }
        }
    }
    //存在，就直接返回
    return (T) instance;
}
```

创建扩展实例，`createExtension(name);`：

```
private T createExtension(String name) {
    //getExtensionClasses加载当前Extension的所有实现
    //上面已经解析过，返回的是一个Map，键是name，值是name对应的Class
    //根据name查找对应的Class
    //比如name是dubbo，Class就是com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol
    Class<?> clazz = getExtensionClasses().get(name);
    //如果这时候class还不存在，说明在所有的配置文件中都没找到定义，抛异常
    if (clazz == null) {
        throw findException(name);
    }
    try {
        //从已创建实例缓存中获取
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        //不存在的话就创建一个新实例，加入到缓存中去
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        //这里实例就是具体实现的实例了比如是DubboProtocol的实例
        //属性注入，在上面已经解析过了，根据实例中的setXxx方法进行注入
        injectExtension(instance);
        //Wrapper的包装
        //cachedWrapperClasses存放着所有的Wrapper类
        //cachedWrapperClasses是在加载扩展实现类的时候放进去的
        //Wrapper类的说明在最上面扩展点自动包装（AOP）
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && wrapperClasses.size() > 0) {
            for (Class<?> wrapperClass : wrapperClasses) {
            	//比如在包装之前的instance是DubboProtocol实例
                //先使用构造器来实例化当前的包装类
                //包装类中就已经包含了我们的DubboProtocol实例
                //然后对包装类进行injectExtension注入，注入过程在上面
                //最后返回的Instance就是包装类的实例。
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        //这里返回的是经过所有的包装类包装之后的实例
        return instance;
    } catch (Throwable t) { }
}
```
获取的Extension是经过层层包装的扩展实现，然后就是调用经过包装的refer方法了，这就到了具体的实现中的方法了。

到此为止调用`refprotocol.refer(Class<T> type, URL url))`方法的过程也解析完了。

关于getActivateExtension方法的解析，等下再添加。

