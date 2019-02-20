假设有这样一段示例代码：

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

可以看到每个SPI扩展的`ExtensionLoader`的实例只有一个，缓存的key就是具体SPI接口类型，比如`com.alibaba.dubbo.rpc.Protocol`作为key。

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

上面示例代码执行后，第一次到这里，type是`com.alibaba.dubbo.rpc.Protocol`，所以这里会先执行`ExtensionLoader.getExtensionLoader(ExtensionFactory.class)`，然后执行`getAdaptiveExtension()`。

也就是说如果是第一次执行获取Protocol类型的ExtensionLoader的实例的话，会先获取ExtensionFactory类型的ExtensionLoader实例。为什么要先获取ExtensionFactory类型的ExtensionLoader的实例呢？因为ExtensionFactory是用来生成扩展点具体实现的工厂，这里暂时先到这里，后面会再说ExtensionFactory相关的东西。

获取完了ExtensionFactory类型的ExtensionLoader后，紧接着调用getAdaptiveExtension()方法来获取一个自适应的ExtensionFactory实例，获取自适应AdaptiveExtensionFactory实例的原因是ExtensionFactory会有多个实现，这样可以在运行时来决定调用哪个具体实现，而不是直接写死使用哪个具体实现。

ExtensionFactory的具体实现有三个：

- AdaptiveExtensionFactory
- SpiExtensionFactory
- SpringExtensionFactory

其中AdaptiveExtensionFactory注解了`@Adaptive`注解，是ExtensionFactory这个SPI接口的自适应实现，如果在运行时需要获取一个ExtensionFactory的实现时，会调用AdaptiveExtensionFactory来进行动态获取。

说明一下，一个扩展点最多只能有一个自适应实现，也就是一个扩展点的具体实现类最多只能有一个可以在类级别上注解`@Adaptive`。如果一个扩展点没有任何一个实现在类级别上注解`@Adaptive`，那么dubbo会在运行时动态生成一个自适应实现类，比如Protocol的具体实现类就没有任何一个有在类级别上注解了`@Adaptive`，dubbo会自动生成一个名字是`Protocol$Adpative`的自适应实现类。

上面的步骤完成了获取Protocol类型的ExtensionLoader的实例，同时也完成了ExtensionFactory类型的ExtensionLoader实例的加载，同时也生成了ExtensionFactory的自适应实现，接下来继续往下走：

```
Protocol dubboProtocol = extensionLoader.getExtension("dubbo");
```

获取了Protocol类型的ExtensionLoader实例后，就可以根据名字来加载具体的实现类了，Protocol的具体实现类有：

- DubboProtocol
- HessianProtocol
- HttpProtocol
- ThriftProtocol
- InjvmProtocol
- RmiProtocol
- WebServiceProtocol
- RegistryProtocol
- RedisProtocol
- MemcachedProtocol
- 一些Wrapper类

可以看到Protocol有很多具体的实现，根据使用协议的不同，可以动态选择具体使用哪一个Protocol实现。

继续看`getExtension()`方法：

```
public T getExtension(String name) {
	if (name == null || name.length() == 0)
		throw new IllegalArgumentException("Extension name == null");
	// 获取默认实现
	if ("true".equals(name)) {
		return getDefaultExtension();
	}
	// 从缓存获取
	Holder<Object> holder = cachedInstances.get(name);
	if (holder == null) {
		cachedInstances.putIfAbsent(name, new Holder<Object>());
		holder = cachedInstances.get(name);
	}
	Object instance = holder.get();
	if (instance == null) {
		synchronized (holder) {
			instance = holder.get();
			if (instance == null) {
				// 缓存不存在，创建实例
				instance = createExtension(name);
				// 加入缓存
				holder.set(instance);
			}
		}
	}
	return (T) instance;
}
```

该方法是根据指定的名字来获取具体的扩展点的实现的实例，比如我们这里传的name是dubbo，就会获取DubboProtocol的实例，具体步骤如下：

1. 校验
2. 如果name是true，就获取默认扩展点的实现实例
3. 从缓存中获取扩展点实现实例
4. 如果缓存中不存在，就根据name创建具体的扩展点实现实例
5. 返回name对应的具体扩展点实现的实例

获取默认扩展点实现实例暂时不说，先看下cachedInstances缓存：

```
private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<String, Holder<Object>>();
```

这里缓存了扩展点具体实现的实例，key是扩展点的名字，比如DubboProtocol的实例，key就是dubbo，value是DubboProtocol的实例，Holder中持有DubboProtocol的实例。

接下来看根据name创建具体扩展点实现实例的方法`createExtension(name)`方法，该方法的代码如下：

```
private T createExtension(String name) {
	/**
	 * getExtensionClasses加载当前扩展点的所有实现
	 * 比如：
	 * 我们在使用ExtensionLoader.getExtensionLoader(Protocol.class)
	 * 获取Protocol的ExtensionLoader的时候，就已经设置了当前ExtensionLoader
	 * 的类型是Protocol的，所以这里获取的时候就是Protocol的所有实现。
	 *
	 * 获取到所有的实现之后，getExtensionClasses()返回的是Map<String, Class<?>>
	 */
	Class<?> clazz = getExtensionClasses().get(name);
	if (clazz == null) {
		throw findException(name);
	}
	try {
		/**
		 * 从缓存中获取已经创建的扩展点的实现的实例
		 * 如果还没有，就根据Class通过反射来创建具体的实例，
		 * 并放到缓存中去
		 */
		T instance = (T) EXTENSION_INSTANCES.get(clazz);
		if (instance == null) {
			EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
			instance = (T) EXTENSION_INSTANCES.get(clazz);
		}
		/**
		 * 向实例中注入依赖的扩展
		 * 如果一个扩展点A依赖了其他的扩展点B，并且有setter方法
		 * 就会执行将扩展点B注入扩展点A的操作
		 */
		injectExtension(instance);
		/**
		 * 如果扩展点有包装类，将扩展点进行包装
		 * 包装后如果也依赖了其他扩展点，也需要注入其他扩展点
		 */
		Set<Class<?>> wrapperClasses = cachedWrapperClasses;
		if (wrapperClasses != null && wrapperClasses.size() > 0) {
			for (Class<?> wrapperClass : wrapperClasses) {
				instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
			}
		}
		return instance;
	} catch (Throwable t) {
		throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
				type + ")  could not be instantiated: " + t.getMessage(), t);
	}
}
```

该方法根据扩展点的名字来创建具体扩展点实现的实例，具体步骤如下：

1. 通过`getExtensionClasses()`方法将当前扩展点的所有的实现类进行加载，如果是`@Adaptive`注解的自适应实现类，则放到cachedAdaptiveClass缓存中；如果是包装类，则放到cachedWrapperCalsses缓存中。经过这一步，扩展点的所有实现都已经解析加载。
2. 根据名字获取到具体的某一个扩展点实现类，并去EXTENSION_INSTANCES缓存中查询是不是有实例，如果没有的话，就使用反射创建一个实例。
3. 如果该实例中依赖了其他的扩展点（需要有setter方法），需要将依赖的扩展点进行注入。
4. 如果扩展点有包装类，则将扩展点进行包装，如果包装后，也依赖了其他的扩展点（需要有setter方法），需要将依赖的扩展点进行注入。
5. 返回注入和包装后的扩展点实现的实例，在我们的这个例子中返回的不是DubboProtocol实例了，而是经过了ProtocolFilterWrapper和ProtocolListenerWrapper包装后的实例。

总体的流程就算说完了，已经获取到了名字为dubbo的Protocol的实现的实例，接下来的执行最后一行代码，得到结果：

```
System.out.println(dubboProtocol.getDefaultPort());
```

接下来我们看看`getExtensionClasses()`方法具体做了什么，该方法是用来加载当前扩展点的所有实现的class的，具体代码如下：

```
private Map<String, Class<?>> getExtensionClasses() {
	/**
	 * 先从缓存中获取，不存在的话就调用loadExtensionClasses进行加载
	 * cachedClasses缓存中存储了当前扩展点所有的实现类
	 */
	Map<String, Class<?>> classes = cachedClasses.get();
	if (classes == null) {
		synchronized (cachedClasses) {
			classes = cachedClasses.get();
			if (classes == null) {
				/**
				 * 如果没有加载Extension的实现，进行扫描加载，完成后缓存起来
				 * 每个扩展点，其实现的加载只会执行一次
				 * 例如，如果Protocol的某个具体实现加载出错了，没有放到缓存中去
				 * 后面再使用，也不会再进行加载了。
				 */
				classes = loadExtensionClasses();
				// 缓存起来
				cachedClasses.set(classes);
			}
		}
	}
	return classes;
}
```

这里也只是尝试从缓存中获取，如果缓存中不存在的话，就进行具体的加载逻辑。但是这里有个点要注意，一个扩展点的的实现类加载只会执行一次。

继续往下走就是真正的加载扩展点的实现逻辑了，代码如下：

```
private Map<String, Class<?>> loadExtensionClasses() {
	final SPI defaultAnnotation = type.getAnnotation(SPI.class);
	if(defaultAnnotation != null) {
		// 当前扩展点的默认实现名字，如果有的话进行缓存
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

	// 从配置文件中加载扩展实现类
	Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
	// 从META-INF/dubbo/internal目录下加载
	loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
	// 从META-INF/dubbo/目录下加载
	loadFile(extensionClasses, DUBBO_DIRECTORY);
	// 从META-INF/services/下加载
	loadFile(extensionClasses, SERVICES_DIRECTORY);
	return extensionClasses;
}
```

这里面逻辑也挺简单的，先获取扩展点的默认名字，如果有的话进行缓存；然后就从配置文件中加载具体的实现类了，加载的位置有三个，请参照代码里的注释。

具体的从配置文件中加载的代码，就不在贴出来了，太长了。说下大概的逻辑：

1. 组装配置文件名字，加载配置文件，遍历文件中每一行进行处理。
2. 加载配置文件中配置的实现类。
3. 如果是注解了`@Adaptive`注解的实现类，加入到cachedAdaptiveClass缓存中。
4. 如果是包装类型的实现类，加入到cachedWrapperClasses缓存中。
5. 如果是除了上面两种的类，放到extensionClasses这个map中，用于在上层返回。

