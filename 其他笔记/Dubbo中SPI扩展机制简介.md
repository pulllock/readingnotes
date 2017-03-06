
dubbo的SPI机制类似与Java的SPI，Java的SPI会一次性的实例化所有扩展点的实现，有点显得浪费资源。

- dubbo的扩展机制可以方便的获取某一个想要的扩展实现，每个实现都有自己的name，可以通过name找到具体的实现。
- 每个扩展点都有一个@Adaptive实例，用来注入到依赖这个扩展点的某些类中，运行时通过url参数去动态判断最终选择哪个Extension实例用。
- dubbo的SPI扩展机制增加了对扩展点自动装配（类似IOC）和自动包装（类似AOP）的支持。
- 标注了@Activate的扩展点实现类，可以通过getActivateExtension方法获取，主要用来实现一次获取多个扩展点实现，getExtension一次只能获取一个实现。

## 扩展点自动装配（IOC）功能：

```
接口A，实现类A1，A2
接口B，实现类B1，B2

实现类A1含有setB()方法，就会自动的注入一个B的实现类，但是不是注入B1和B2，而是注入一个动态生成的B的实现类B$Adpative，该实现能够根据参数的不同，自动选择B1或者B2来完成相应功能。

```

## 扩展点自动包装（AOP）功能：

其实是对扩展点采用装饰器模式进行增强。

```
接口A还有另外一个实现者：AWrapper1，此实现者有如下构造器：
private A a;
AWrapper1(A a){
  this.a = a;
}
```

当我们再获取接口A的实现类的时候，就已经被AWrapper1包装过了，我们得到的就是包装过得类。

## ExtensionLoader

ExtensionLoader是dubbo的SPI机制的查找服务实现的工具类，类似与Java的ServiceLoader，可做类比。dubbo约定扩展点配置文件放在classpath下的`META-INF/dubbo`目录下，配置文件名为接口的全限定名，配置文件内容为`配置名=扩展实现类的全限定名`。

重点解析下ExtensionLoader这个类。dubbo的扩展点使用单一实例去加载，缓存在ExtensionLoader中。每一个ExtensionLoader实例仅负责加载特定SPI扩展的实现，想要获得某个扩展的实现，首先要获得该扩展对应的ExtensionLoader实例。

以Protocol为例进行分析扩展点的加载：

```
//这样使用，加载Protocol扩展点
Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```
第一步，getExtensionLoader(Protocol.class)，根据要加载的接口Protocol，创建出一个ExtensionLoader实例，加载完的实例会被缓存起来：

```
//缓存所有的扩展加载实例
//比如这里加载Protocol.class，就以Protocol.class作为key，以新创建的ExtensionLoader作为value
//每一个要加载的扩展点只会对应一个ExtensionLoader实例，也就是只会存在一个Protocol.class在缓存中
private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();
```

getExtensionLoader(Protocol.class)这一步没有进行任何的加载工作，只是获得了一个ExtensionLoader的实例。加载是在调用getAdaptiveExtension()方法中进行的：

```
getAdaptiveExtension()-->
                createAdaptiveExtension()-->
                                getAdaptiveExtensionClass()-->
                                                getExtensionClasses()-->
                                                                loadExtensionClasses()

```
loadExtensionClasses()这个方法中加载扩展点的实现工作。

```
private Map<String, Class<?>> loadExtensionClasses() {
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if(defaultAnnotation != null) {
        //当前Extension的默认实现名字
        String value = defaultAnnotation.value();
        if(value != null && (value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if(names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                        + ": " + Arrays.toString(names));
            }
            if(names.length == 1) cachedDefaultName = names[0];
        }
    }

    //从配置文件中加载扩展实现类
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
首先解析Protocol扩展点的默认的实现名字，Protocol接口上面注释的默认实现名字是dubbo：

```
@SPI("dubbo")
public interface Protocol
```
获取到默认实现名字之后缓存到cachedDefaultName属性中。

获取完默认实现的名字之后，就开始从配置文件中加载扩展的具体实现类，加载解析配置文件的位置为：

```
META-INF/dubbo/internal/
META-INF/dubbo/
META-INF/services/
```
对于Protocol来说加载的文件是`com.alibaba.dubbo.rpc.Protocol`文件，文件的内容是（有好几个同名的配置文件，这里直接把内容全部写在了一起）：

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
对于每个文件都分别一行一行读取解析。对于上面列出来的所有的Protocol扩展的实现类解析总共分成三种情况来处理：

1. 如果这个实现类含有Adaptive注解，则将这个class赋值给cachedAdaptiveClass。
2. 尝试获取带有对应接口参数的构造器，如果能够获取到，说明这个实现类是一个Wrapper类，会赋值给cachedWrapperClasses。
3. 如果上面都没有，则获取实现类上的Extension注解，把注解定义的name作为key，存储到cachedClasses中。

到这里解析就已经完成了，返回getAdaptiveExtensionClass中，在调用完getExtensionClasses()之后，会首先检查是不是已经有@Adaptive注解的类被解析并加入到缓存中了，如果有就直接返回，没有就动态创建一个：

```
private Class<?> getAdaptiveExtensionClass() {
    //加载当前Extension的所有实现，如果有@Adaptive类型，会赋值给cachedAdaptiveClass
    getExtensionClasses();
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    //没有找到Adaptive类型的实现，动态创建一个
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

动态创建AdaptiveExtensionClass：

```
private Class<?> createAdaptiveExtensionClass() {
	//组装动态创建扩展点类的代码
    String code = createAdaptiveExtensionClassCode();
    //获取到应用的类加载器
    ClassLoader classLoader = findClassLoader();
    //获取编译器
    //dubbo默认使用javassist
    com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    return compiler.compile(code, classLoader);
}
```

@Adaptive注解在实现类和方法上的区别：

如果注解在方法上，当调用getAdaptiveExtension()方法获取适配扩展点的时候，会先通过前面的过程生成源码，然后通过编译器生成class进行加载。但是上面代码中的Compiler的获取也是通过getAdaptiveExtension()方法获取的，这样不就成了死循环了吗？看下源码：

```
@Adaptive
public class AdaptiveCompiler implements Compiler
```
在Compiler的实现类AdaptiveCompiler上有注解@Adaptive，同时在getAdaptiveExtensionClass方法中：

```
private Class<?> getAdaptiveExtensionClass() {
    //加载当前Extension的所有实现，如果有@Adaptive类型，会赋值给cachedAdaptiveClass
    getExtensionClasses();
    //上面一步加载的时候，如果有注解了@Adaptive的类，cachedAdaptiveClass就会被赋值，然后直接返回
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    //没有找到Adaptive类型的实现，动态创建一个
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```
注解了@Adaptive的类，会在这里直接返回，不会继续去动态创建。好像在dubbo中只有AdaptiveCompiler和AdaptiveExtensionFactory这两个类注解了@Adaptive注解，其他都是方法上注解。

创建完Adaptive扩展之后，就需要注入了。

返回createAdaptiveExtension方法中：`injectExtension((T) getAdaptiveExtensionClass().newInstance())`，在getAdaptiveExtensionClass获取完并创建新实例之后，就会进行依赖注入injectExtension。

injectExtension方法：

```
private T injectExtension(T instance) {
    try {
        if (objectFactory != null) {
            for (Method method : instance.getClass().getMethods()) {
                //处理set方法
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
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {//说明set方法的参数是扩展点类型，进行注入
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("fail to inject via method " + method.getName()
                                + " of interface " + type.getName() + ": " + e.getMessage(), e);
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```
