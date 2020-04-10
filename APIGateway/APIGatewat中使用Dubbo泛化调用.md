APIGateway需要调用各个业务系统的接口，但是不可能作为消费者依赖所有系统的接口jar包，可以使用Dubbo的泛化调用功能来实现。APIGateway作为消费者，连接到注册中心，拿到相应接口后可以使用泛化调用。

泛化调用比较简单，可以直接参考dubbo官方文档：[Dubbo的泛化调用](http://dubbo.apache.org/zh-cn/blog/dubbo-generic-invoke.html)

示例代码：

```java
public class GenericInvokeDubbo {

    public final static String PROTOCOL = "zookeeper";

    public final static String REGISTRY_ADDRESS = "127.0.0.1:2181";

    public final static String APP_NAME = "app-name";

    private Map<String, ReferenceConfig> referenceConfigMap = new ConcurrentHashMap<String, ReferenceConfig>();

    public Object invokeService(String interfaceClass, String method, String[] paramTypes, Object[] params) {
        ReferenceConfigCache cache = null;
        ReferenceConfig<GenericService> referenceConfig = null;
        try {
            referenceConfig = referenceConfigMap.get(interfaceClass);
            if (referenceConfig == null) {
                referenceConfig = new ReferenceConfig<>();

                ApplicationConfig application = new ApplicationConfig();
                application.setName(APP_NAME);
                referenceConfig.setApplication(application);

                RegistryConfig registry = new RegistryConfig();
                registry.setProtocol(PROTOCOL);
                registry.setAddress(REGISTRY_ADDRESS);
                referenceConfig.setRegistry(registry);

                ConsumerConfig consumerConfig = new ConsumerConfig();
                consumerConfig.setTimeout(5000);
                consumerConfig.setRetries(0);
                referenceConfig.setConsumer(consumerConfig);

                referenceConfig.setGeneric(true);
                // referenceConfig.setVersion();
                referenceConfig.setInterface(interfaceClass);
                referenceConfigMap.put(interfaceClass, referenceConfig);
            }
            cache = ReferenceConfigCache.getCache();
            GenericService genericService = cache.get(referenceConfig);
            return genericService.$invoke(method, paramTypes, params);
        } catch (IllegalStateException e) {
            referenceConfigMap.remove(interfaceClass);
            if (cache != null) {
                cache.destroy(referenceConfig);
            }

            e.printStackTrace();
            return null;
        }
        catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

