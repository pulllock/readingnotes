
dubbo的SPI机制类似与Java的SPI，Java的SPI会一次性的实例化所有扩展点的实现，有点显得浪费资源。dubbo的SPI扩展机制增加了对扩展点IOC和AOP的支持。

ExtensionLoader是dubbo的SPI机制的查找服务实现的工具类，类似与Java的ServiceLoader，可做类比。dubbo约定扩展点配置文件放在classpath下的`META-INF/dubbo`目录下，配置文件名为接口的全限定名，配置文件内容为`配置名=扩展实现类的全限定名`。

重点解析下ExtensionLoader这个类。dubbo的扩展点使用单一实例去加载，缓存在ExtensionLoader中。每一个ExtensionLoader实例仅负责加载特定SPI扩展的实现，想要获得某个扩展的实现，首先要获得该扩展对应的ExtensionLoader实例。

```
public class ExtensionLoader<T> {

    private static final Logger logger = LoggerFactory.getLogger(ExtensionLoader.class);
    //
    private static final String SERVICES_DIRECTORY = "META-INF/services/";

    private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";

    private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/";

    private static final Pattern NAME_SEPARATOR = Pattern.compile("\\s*[,]+\\s*");

    private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();

    private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<Class<?>, Object>();

    // ==============================

    private final Class<?> type;

    private final ExtensionFactory objectFactory;

    private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<Class<?>, String>();

    private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<Map<String,Class<?>>>();

    private final Map<String, Activate> cachedActivates = new ConcurrentHashMap<String, Activate>();

    private volatile Class<?> cachedAdaptiveClass = null;

    private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<String, Holder<Object>>();

    private String cachedDefaultName;

    private final Holder<Object> cachedAdaptiveInstance = new Holder<Object>();
    private volatile Throwable createAdaptiveInstanceError;

    private Set<Class<?>> cachedWrapperClasses;

    private Map<String, IllegalStateException> exceptions = new ConcurrentHashMap<String, IllegalStateException>();

}
```
