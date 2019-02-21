从两个示例代码，介绍dubbo的SPI的使用以及相关源码分析，分析了获取扩展实现和获取自适应扩展点实现的源码，最后简单说了下ExtensionFactory的流程，看完就可以理解为什么dubbo是自包含的了。从上往下看，再回头看，应该能看明白，文章比较长，希望能耐心读下去。如果有错误的地方希望能指出来，我也理解不是太完整或者表述不是太明白。

# ExtensionLoader使用以及简单流程分析

假设有这样一段示例代码：

```java
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

```java
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

## ExtensionLoader缓存

前面的校验可以参考注释，这里先说下缓存`EXTENSION_LOADERS`：

```java
private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();
```

可以看到每个SPI扩展的`ExtensionLoader`的实例只有一个，缓存的key就是具体SPI接口类型，比如`com.alibaba.dubbo.rpc.Protocol`作为key。

## ExtensionLoader实例化

`new ExtensionLoader<T>(type)`这里做了什么？

```java
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

# 使用ExtensionLoader获取扩展点实现

上面的步骤完成了获取Protocol类型的ExtensionLoader的实例，同时也完成了ExtensionFactory类型的ExtensionLoader实例的加载，同时也生成了ExtensionFactory的自适应实现，接下来继续往下走：

```java
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

```java
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

## 扩展点实现的实例缓存

获取默认扩展点实现实例暂时不说，先看下cachedInstances缓存：

```java
private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<String, Holder<Object>>();
```

这里缓存了扩展点具体实现的实例，key是扩展点的名字，比如DubboProtocol的实例，key就是dubbo，value是DubboProtocol的实例，Holder中持有DubboProtocol的实例。

# 创建扩展点实现实例

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

```java
System.out.println(dubboProtocol.getDefaultPort());
```

# 加载扩展点实现类的Class

接下来我们看看`getExtensionClasses()`方法具体做了什么，该方法是用来加载当前扩展点的所有实现的class的，具体代码如下：

```java
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

```java
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

# 扩展点依赖注入

我们在返回上面，还又一点没说，就是依赖注入的功能injectExtension的代码：

```java
private T injectExtension(T instance) {
	try {
		// 在获取第一个扩展点的ExtensionLoader的实例的时候，objectFactory就被实例化了，是AdaptiveExtensionFactory
		if (objectFactory != null) {
			// 遍历要注入的实例的方法
			for (Method method : instance.getClass().getMethods()) {
				// 只处理set方法，比如setA，就是要把A注入到instance中
				if (method.getName().startsWith("set")
						&& method.getParameterTypes().length == 1
						&& Modifier.isPublic(method.getModifiers())) {
					// set方法参数类型
					Class<?> pt = method.getParameterTypes()[0];
					try {
						// setter方法对应的属性名，也就是扩展点接口名称
						String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
						/**
						 * objectFactory是AdaptiveExtensionFactory实例
						 * 比如这里的pt是com.alibaba.dubbo.rpc.Protocol，property是protocol
						 * objectFactory就会根据这两个参数去获取Protocol对应的扩展实现的实例
						 */
						Object object = objectFactory.getExtension(pt, property);
						// 获取到了setter方法的参数的实现，可以进行注入
						if (object != null) {
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

依赖注入的代码也很简单，就是实例化要注入的类，然后反射调用set方法注入实例中去。

# 自适应扩展点使用

到这里，使用指定名称加载扩展点实现的流程就分析完了，但是这种直接指定扩展点名字的方式却不是我们主要使用的方式。可以想象一下，dubbo是可以配置多协议的，也就是可以同时配置比如dubbo、rmi等协议。如果我们使用了多协议的话，那dubbo是怎么做的呢？我们可以想到最简单的方法就是有一个转发器，用来根据实际请求中配置的协议来使用不同的实现来处理，下面可以写个伪代码：

```java
public class ProtocolDispatcher implements Protocol {
    
    public void refer(String name) {
        if (name.equals("dubbo")) {
            Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("dubbo");
        } else if(name.equals("rmi")) {
             Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("rmi");
        }
    }
}
```

实际上dubbo中没有这样的代码，但实际上也差不多类似这样的方式来处理的，我们看下实际在dubbo中的使用方式：

```java
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

可以看到第一步还是先获取Protocol类型的ExtensionLoader的实例，这个过程跟最上面的获取ExtensionLoader实例的过程是一样的，接下来这一步`getAdaptiveExtension()`就跟我们之前的示例不一样了，这是获取自适应扩展的方法。

自适应扩展是不是很熟悉，上面我们也说过自适应，可以回头先去看下大概情况。首先说下获取自适应扩展是干嘛的？其实就是做到上面那个伪代码的转发器功能。

## 自适应扩展点动态生成的代码

当调用了上面`getAdaptiveExtension()`方法后，dubbo会动态生成如下代码：

```java
package com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {
    public void destroy() {
        throw new UnsupportedOperationException(
            "method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!"
        );
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException(
            "method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!"
        );
    }

    public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.Invoker {
        if (arg0 == null) 
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        
        if (extName == null)
            throw new IllegalStateException(
                "Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])"
            );
        
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) 
            ExtensionLoader
                .getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class)
                .getExtension(extName);
        return extension.export(arg0);
    }

    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws java.lang.Class {
        if (arg1 == null) 
            throw new IllegalArgumentException("url == null");
        
        com.alibaba.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        
        if (extName == null)
            throw new IllegalStateException(
                "Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])"
            );
        
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) 
            ExtensionLoader
                .getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class)
                .getExtension(extName);
        return extension.refer(arg0, arg1);
    }
}
```

当我们调用`protocol.xxxx()`方法的时候，其实就是调用动态生成的`Protocol$Adaptive`这个类的方法，这里面的逻辑其还是就跟我们的伪代码差不多了，根据url中传入的Protocol名字，通过`getExtension(extName)`方法获取实际的扩展点实现实例。

# 自适应扩展点的获取

接下来就看下获取自适应扩展的源码：

```java
public T getAdaptiveExtension() {
	// 先从自适应实例缓存中查找实例对象
	Object instance = cachedAdaptiveInstance.get();
	// 缓存中不存在
	if (instance == null) {
		if(createAdaptiveInstanceError == null) {
			synchronized (cachedAdaptiveInstance) {
				// 获取锁之后再检查一次缓存中是不是已经存在
				instance = cachedAdaptiveInstance.get();
				if (instance == null) {
					try {
						// 缓存中没有，就创建新的AdaptiveExtension实例
						instance = createAdaptiveExtension();
						// 新实例加入缓存
						cachedAdaptiveInstance.set(instance);
					} catch (Throwable t) {
						createAdaptiveInstanceError = t;
						throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
					}
				}
			}
		}
		else {
			throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
		}
	}

	return (T) instance;
}
```

这边还是老套路，先从缓存中获取，如果缓存中不存在，就创建自适应扩展实例，继续看`createAdaptiveExtension()`方法：

```java
private T createAdaptiveExtension() {
	try {
		/**
		 * 先通过getAdaptiveExtensionClass获取自适应扩展类的Class
		 * 然后通过反射获取实例
		 * 最后如果自适应扩展依赖了其他的扩展点，就进行扩展点注入
		 */
		return injectExtension((T) getAdaptiveExtensionClass().newInstance());
	} catch (Exception e) {
		throw new IllegalStateException("Can not create adaptive extenstion " + type + ", cause: " + e.getMessage(), e);
	}
}
```

这里的逻辑跟`createExtension()`差不多，大概步骤：

- 先通过`getAdaptiveExtensionClass()`方法获取自适应扩展类的Class
- 然后通过反射获取实例
- 最后如果自适应扩展类实例依赖了其他的扩展点，就进行扩展点的注入

## 获取自适应扩展点类的Class

首先看下获取自适应扩展类的Class方法：

```java
private Class<?> getAdaptiveExtensionClass() {
	/**
	 * getExtensionClasses加载当前扩展点的所有实现
	 * 比如：
	 * 我们在使用ExtensionLoader.getExtensionLoader(Protocol.class)
	 * 获取Protocol的ExtensionLoader的时候，就已经设置了当前ExtensionLoader
	 * 的类型是Protocol的，所以这里获取的时候就是Protocol的所有实现。
	 *
	 * 获取到所有的实现之后，getExtensionClasses()返回的是Map<String, Class<?>>
	 *
	 * 另外需要说的是，如果扩展点的实现注解了类级别的@Adaptive注解，
	 * 这些实现的Class加载完后会赋值给cachedAdaptiveClass缓存。如果扩展点的实现
	 * 是包装类，这些实现的Class加载完后会放到cachedWrapperClasses缓存中。
	 * 其他的正常的扩展点的实现都会放到Map<String, Class<?>>中返回。
	 *
	 * 目前只有AdaptiveExtensionFactory和AdaptiveCompiler两个实现类是被注解了@Adaptive
	 * 也就是说这两个就是自适应扩展，如果要加载ExtensionFactory和Compiler的自适应扩展
	 * 不需要使用自动生成代码，而是直接使用两个实现类就可以了。
	 * 其他的扩展点如果想要获取自适应扩展实现，就需要继续往下走，使用生成的Xxx$Adaptive代码。
	 *
	 * 一个扩展点有且只有一个自适应扩展点，要么是内置的两个AdaptiveExtensionFactory和AdaptiveCompiler，
	 * 要么是生成的Xxx$Adaptive
	 */
	getExtensionClasses();
	/**
	 * 自适应扩展实现，在上面一步加载的时候，就会被加载缓存起来
	 * 只会执行一次，后面再获取的时候，就是获取缓存起来的这个。
	 */
	if (cachedAdaptiveClass != null) {
		return cachedAdaptiveClass;
	}
	// 没有缓存自适应扩展实现，就动态创建一个
	return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

获取自适应扩展类的过程参考上面代码的注释即可，继续往下说创建自适应扩展类的方法`createAdaptiveExtensionClass()`：

```java
private Class<?> createAdaptiveExtensionClass() {
	/**
	 * 根据具体的接口来生成自适应扩展类的代码
	 * 比如Protocol就会生成Protocol$Adaptive为名字的类的代码
	 */
	String code = createAdaptiveExtensionClassCode();
	// 获取类加载器
	ClassLoader classLoader = findClassLoader();
	// 获取Compiler的自适应扩展，获取到的是AdaptiveCompiler实例
	com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
	// 如果我们没有指定名字，默认使用javassist
	return compiler.compile(code, classLoader);
}
```

这里大概的步骤是：

- 生成自适应扩展类的代码。
- 获取类加载器。
- 获取自适应的Compiler的扩展实现，获取到的AdaptiveCompiler实例，这个在上面已经说过了。
- 最后使用具体的Compiler进行生成代码的编译。

这里只看第一步，生成自适应扩展类的代码这步，这里代码有点长，不在此贴出来了，参考我的github上ExtensionLoader的源码注释[ExtensionLoader.java](https://github.com/dachengxi/Dubbo2.5.4SourceCode/blob/master/dubbo-common/src/main/java/com/alibaba/dubbo/common/extension/ExtensionLoader.java)。

# @Adaptive注解

这里说下`@Adaptive`注解，有两种地方使用这个注解：

- 使用在实现类上
- 使用在接口的方法上

这两种不能重复使用。如果用在实现类上，一个扩展点的实现类有且只能有一个类使用此注解，比如ExtensionFactory的实现类AdaptiveExtensionFactory使用了此注解，这个类本身就是一个自适应扩展类了；如果用在接口的方法上，表示dubbo框架会在生成该接口的自适应扩展类的时候，生成该方法的代码，如果方法没有添加此注解，则生成抛出不支持异常的代码。

# ExtensionFactory

到这里获取扩展和获取自适应扩展就已经说完了，接下来可以把最上面留下的ExtensionFactory相关的加载流程说下了，每个ExtensionLoader实例中都会有一个objectFactory实例，而objectFactory实例的赋值都是在ExtensionLoader的构造方法中：

```java
private ExtensionLoader(Class<?> type) {
	this.type = type;
	/**
	 * 对于扩展类型是ExtensionFactory的，设置为null
	 * getAdaptiveExtension方法获取一个运行时自适应的扩展类型
	 * 每个Extension只能有一个@Adaptive类型的实现，如果么有，dubbo会自动生成一个类
	 * objectFactory是一个ExtensionFactory类型的属性，主要用于加载扩展的实现
	 */

	objectFactory = (
			type == ExtensionFactory.class ?
					null :
					ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()
	);
}
```

可以看到ExtensionFactory的实例获取也是通过扩展点自适应来获取到的，获取到的实例是AdaptiveExtensionFactory。而在AdaptiveExtensionFactory实例化的时候，会通过SPI机制加载所有的ExtensionFactory的实现：

```java
public AdaptiveExtensionFactory() {
	ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
	List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
	for (String name : loader.getSupportedExtensions()) {
		// 保存所有ExtensionFactor y的实现
		list.add(loader.getExtension(name));
	}
	factories = Collections.unmodifiableList(list);
}
```

使用objectFactory获取扩展的时候，是调用AdaptiveExtensionFactory的getExtension方法，该方法会遍历所有的ExtensionFactory的实现的getExtension方法：

```java
public <T> T getExtension(Class<T> type, String name) {
	// 依次遍历各个ExtensionFactory实现的getExtension方法
	// 找到Extension后立即返回，没找到返回null
	for (ExtensionFactory factory : factories) {
		T extension = factory.getExtension(type, name);
		if (extension != null) {
			return extension;
		}
	}
	return null;
}
```

共两种实现SpiExtensionFactory和SpringExtensionFactory，如果在任何一个实现中找到了扩展点实现，就返回结束了。

dubbo是自包含的，这个概念通过上面的解析也应该不难理解了。



