假设有这样一段代码：

```
public static void main(String[] args) {
        ExtensionLoader<Protocol> extensionLoader = ExtensionLoader.getExtensionLoader(Protocol.class);

        Protocol dubboProtocol = extensionLoader.getExtension("dubbo");
        System.out.println(dubboProtocol.getDefaultPort());
    }
```

我们先通过`ExtensionLoader.getExtensionLoader(Protocol.class)`获取`ExtensionLoader`实例，然后通过`getExtension("dubbo")`获取到具体的`Protocol`实现`DubboProtocol`。

首先看下获取`ExtensionLoader`实例的过程：

1. 各种校验。
2. 从缓存中获取指定类型的`ExtensionLoader`实例。
3. 如果缓存中不存在的话，就新建一个`ExtensionLoader`实例，并放入缓存。
4. 返回`ExtensionLoader`实例。

这部分源码如下：

```
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    // 扩展点类型不能为空
    if (type == null)
        throw new IllegalArgumentException("Extension type == null");
    // 扩展点类型只能是接口类型的
    if(!type.isInterface()) {
        throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
    }
    // 没有添加@SPI注解
    if(!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type(" + type + 
                ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
    }
    // 先从缓存中获取指定类型的ExtensionLoader
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    // 缓存中不存在
    if (loader == null) {
        /**
         * 创建一个新的ExtensionLoader实例，放到缓存中去
         * 对于每一个扩展，dubbo中只有个对应的ExtensionLoader实例
         */
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```

前面的校验可以参考注释，这里先说下缓存`EXTENSION_LOADERS`：

```
private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();
```

可以看到每个SPI扩展的`ExtensionLoader`的实例只有一个，缓存的key就是具体SPI接口类型，比如`Protocol.class`作为key。

`new ExtensionLoader<T>(type)`这里做了什么？

```
private ExtensionLoader(Class<?> type) {
    this.type = type;
    objectFactory = (
            type == ExtensionFactory.class ?
                    null :
                    ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()
    );
}
```

这里做了两件事：

1. 将当前类型赋值给当前`ExtensionLoader`实例的type字段。
2. 为当前实例创建一个Extension工厂。