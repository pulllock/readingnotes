继续来看下Dubbo的服务提供者启动的过程，重点是说下服务提供者初始化的过程，也就是导出服务的过程。还是以一个示例为入口来进行解析，一个示例是通过Spring来启动Dubbo，另外一个示例是通过Dubbo的Api方式来启动。其中开始的时候会牵涉到Spring的扩展以及在什么时候Spring通知Dubbo进行服务的导出。Spring的扩展请自行查阅相关资料，另外下面会提供一篇Spring扩展点汇总的文章。

# 预备知识

- Spring中扩展点汇总文档链接：[Spring中扩展点汇总](https://cxis.me/2019/02/22/Spring%E4%B8%AD%E6%89%A9%E5%B1%95%E7%82%B9%E6%B1%87%E6%80%BB/)

# 示例以及服务提供者启动过程

## 示例代码

这里只贴出一点重要的代码，完整的代码请参考Github仓库：[DubboTest](https://github.com/dachengxi/DubboTest)

dubbo-provider.xml:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo
       http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!--当前应用信息，name是当前应用的名称，用于注册中心计算依赖关系，消费方和提供方的名称不要一样-->
    <dubbo:application name="dubbo-provider" version="1.0" owner="cheng.xi" organization="china" environment="product" />

    <!--注册中心配置，protocol注册中心的地址协议，address注册中心服务器地址-->
    <dubbo:registry  protocol="zookeeper" address="127.0.0.1:2181" />

    <!--服务提供者协议配置，name协议名称，port端口-->
    <dubbo:protocol name="dubbo" port="20880" />

    <bean id="helloService" class="me.cxis.dubbo.service.impl.HelloServiceImpl" />
    <!--服务提供者暴露服务配置，interface接口名，ref服务对象实现引用-->
    <dubbo:service interface="me.cxis.dubbo.service.HelloService" ref="helloService"/>
</beans>
```

HelloServiceImpl：

```java
public class HelloServiceImpl implements HelloService {
    @Override
    public void sayHello() {
        System.out.println("这里是Provider");
        System.out.println("HelloWorld Provider！");
    }
}
```

启动Provider代码：

```java
public class StartProvider {

    public static void main(String[] args){
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"dubbo-provider.xml"});
        context.start();
        System.out.println("这里是dubbo-provider服务，按任意键退出");
        try {
            System.in.read();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}
```



## ## Provider启动过程

1. 用户启动Spring容器。
2. Spring碰到Dubbo的标签，使用dubbo的标签解析器解析dubbo的配置文件。
3. dubbo的标签解析器解析完成后，会将配置文件封装成ServiceBean，也是一个ServiceConfig。
4. Spring容器会在适当的时候执行ServiceConfig的export方法，分为延迟导出和非延迟导出，下面会说。
5. export方法绑定ip，端口，启动netty服务器。
6. 注册服务到注册中心。

# Spring启动后遇到Dubbo标签

Spring容器启动后，遇到自定义标签，会使用相应的标签解析器进行解析标签，比如遇到了dubbo的service标签后，就会使用`com.alibaba.dubbo.config.spring.schema.DubboBeanDefinitionParser`来解析service标签，对应的bean是`com.alibaba.dubbo.config.spring.ServiceBean`，标签解析完成后Spring容器就会存在一个`ServiceBean<T>`，我们先看下ServiceBean的相关代码：

```java
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener, BeanNameAware {
    ...
}
```

ServiceBean实现了Spring的若干接口，会在容器启动以及Bean实例化的时候依次执行相关方法：

1. BeanNameAware接口，会调用setBeanName方法设置bean的名字。
2. ApplicationContextAware接口，会调用setApplicationContext方法设置上下文。
3. InitializingBean接口，会调用afterPropertiesSet方法，执行逻辑。
4. 实现了ApplicationListener，会调用onApplicationEvent方法，等到容器刷新完成事件发出后，执行逻辑。

## 服务发布的时机

我们来看下服务的发布时机，也就是在什么时候导出服务。dubbo服务导出有根据配置的delay参数不同，有几种情况考虑。第一种是配置了`delay=-1`或者不配置delay，如下代码：

```xml
<dubbo:service delay="-1" />
```

这种情况是延迟到Spring容器初始化完成后，才进行服务导出。即等到容器刷新完成事件完成后，进行服务导出动作。

第二种情况是配置delay大于等于0的值，代码如下：

```xml
<dubbo:service delay="5000" />
```

这种情况下，会在Bean初始化的时候，在调用afterPropertiesSet方法的时候，就开始导出服务，但是需要延迟指定的时间，也就是delay配置的时间，而不需要等到Spring容器初始化完成再进行服务导出。具体看下isDelay()方法的源码：

```java
private boolean isDelay() {
	// 获取service标签中配置的delay属性
	Integer delay = getDelay();
	// 获取provider标签
	ProviderConfig provider = getProvider();
	// 如果service标签中配置的delay为空，就使用provider标签配置的delay
	if (delay == null && provider != null) {
		delay = provider.getDelay();
	}
	/**
	 * 如果支持应用监听器，并且没有配置delay，该方法返回true，这种情况下，需要延迟到
	 * Spring容器刷新完成后，也就是等到容器刷新事件后进行服务暴露。
	 *
	 * 如果支持应用监听器，并且配置了delay，且delay=-1，该方法返回true，这种情况下，需要延迟到
	 * Spring容器刷新完成后，也就是等到容器刷新事件后进行服务暴露。
	 *
	 * 如果支持应用监听器，并且配置了delay，且delay是大于等于0的，该方法返回false，
	 * 这种情况下是在afterPropertiesSet方法中就直接导出服务，但是需要延迟指定的delay时间，
	 * 不需要等到容器刷新事件完成后再导出服务。
	 */
	return supportedApplicationListener && (delay == null || delay.intValue() == -1);
}
```

# 服务导出

上面简单说了下服务导出的时机，不管哪种情况，都是调用export方法进行服务导出的，我们接下来看看export方法的源码分析。服务导出的大概步骤如下：

1. 关于延迟的相关处理。
2. 一些前置检查，属性校验等等工作。
3. 组装url。
4. 导出服务，包括导出到本地和导出到远程服务。这其中也会有很多步骤，接下来会说。
5. 注册服务到注册中心。

## 服务导出前延迟处理

我们一步一步来看，先看下入口export方法，主要是有关延迟的处理：

```
public synchronized void export() {
	/**
	 * ProviderCoonfig对应<dubbo:provider>，服务提供者的默认值，
	 * 如果service和protocol标签没设置相关属性，就会使用这个里面的默认值
	 *
	 * 如果配置了provider且service和protocol中没有配置相关属性，
	 * 就尝试从provider中获取export和delay属性
	 */
	if (provider != null) {
		if (export == null) {
			export = provider.getExport();
		}
		if (delay == null) {
			delay = provider.getDelay();
		}
	}
	// 配置了不允许暴露，就直接返回，不暴露服务
	if (export != null && ! export.booleanValue()) {
		return;
	}
	// 设置了delay的值，启动新线程，sleep delay的时间，再去doExport
	if (delay != null && delay > 0) {
		Thread thread = new Thread(new Runnable() {
			public void run() {
				try {
					Thread.sleep(delay);
				} catch (Throwable e) {
				}
				doExport();
			}
		});
		thread.setDaemon(true);
		thread.setName("DelayExportServiceThread");
		thread.start();
	} else {
		// 没有设置delay时间，直接doExport，开始暴露服务
		doExport();
	}
}
```

这里面逻辑很简单，就是关于可不可以导出服务以及延迟的处理，如果都没问题，就继续导出服务。

## 服务导出前各种默认配置的处理

接下来继续看doExport方法：

```java
protected synchronized void doExport() {
    /**
    * 该方法的代码有点长，主要的作用是检查各种属性
    * 从系统环境变量等地方获取各种配置，来填充默认属性值。
    * 然后处理generic泛化调用的接口、local、stub等
    * 接着是检查application、registry、protocol等配置，
    * 有必要的话，设置默认值等等。
    * 最后调用doExportUrls()来导出服务。
    */
}
```

# 导出服务

服务导出前的准备工作都做完了之后，接下来就是要开始导出服务了，方法是doExportUrls()：

```java
private void doExportUrls() {
	/**
	 * 加载注册中心URL
	 * 组装成类似这种：registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?
	 * application=dubbo-provider&application.version=1.0&dubbo=2.5.3&environment=product&organization=china&
	 * owner=cheng.xi&pid=11806&registry=zookeeper&timestamp=1488808916423
	 * 
	 * 这里是一个List，也就是dubbo允许我们使用多协议和多注册中心来导出服务
	 */
	List<URL> registryURLs = loadRegistries(true);
	// 遍历所有的协议，分别导出
	for (ProtocolConfig protocolConfig : protocols) {
		doExportUrlsFor1Protocol(protocolConfig, registryURLs);
	}
}
```

dubbo允许多协议导出服务，也允许注册多个注册中心，所以loadRegistries方法返回的是一个List，表示多协议。方法的参数true表示这是服务提供者，如果是服务消费者的话传false。使用loadRegistries方法加载完注册中心的url后，就遍历所有协议挨个进行服务的导出。

## 加载注册中心URL

加载注册中心URL的方法是loadRegistries：

```java
protected List<URL> loadRegistries(boolean provider) {
	// 检查注册中心配置，如果不存在抛异常；如果存在填充各种Registry属性
	checkRegistry();
	// 存放注册中心的URL
	List<URL> registryList = new ArrayList<URL>();
	if (registries != null && registries.size() > 0) {
		// 遍历注册中心配置
		for (RegistryConfig config : registries) {
			// 获取配置中的注册中心地址
			String address = config.getAddress();
			if (address == null || address.length() == 0) {
				// 没配置，使用默认的0.0.0.0
				address = Constants.ANYHOST_VALUE;
			}
			// 尝试从系统属性中获取
			String sysaddress = System.getProperty("dubbo.registry.address");
			if (sysaddress != null && sysaddress.length() > 0) {
				address = sysaddress;
			}
			if (address != null && address.length() > 0 
					&& ! RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(address)) {
				Map<String, String> map = new HashMap<String, String>();
				// 配置的application中的属性取出来放到map中
				appendParameters(map, application);
				// 配置的registry中的属性取出来放到map中
				appendParameters(map, config);
				// 服务名
				map.put("path", RegistryService.class.getName());
				// dubbo的版本
				map.put("dubbo", Version.getVersion());
				// 时间戳
				map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
				// pid
				if (ConfigUtils.getPid() > 0) {
					map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
				}
				// map中不存在protocol，说明没有配置procotol
				if (! map.containsKey("protocol")) {
					// 查看有没有名为remote的Registry的实现类
					if (ExtensionLoader.getExtensionLoader(RegistryFactory.class).hasExtension("remote")) {
						map.put("protocol", "remote");
					} else {
						// 没有的话Protocol使用dubbo
						map.put("protocol", "dubbo");
					}
				}
				/**
				 * 解析生成url
				 * zookeeper://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=dubbo-provider&application.version=1.0&
				 * dubbo=2.5.3&environment=product&organization=china&owner=cheng.xi&pid=17268&timestamp=1488986530185
				 * addres可能配置了多个，所有要解析成一个URL列表
				 */
				List<URL> urls = UrlUtils.parseURLs(address, map);
				for (URL url : urls) {
					// URL协议头设置为registry
					url = url.addParameter(Constants.REGISTRY_KEY, url.getProtocol());
					url = url.setProtocol(Constants.REGISTRY_PROTOCOL);
					if ((provider && url.getParameter(Constants.REGISTER_KEY, true))
							|| (! provider && url.getParameter(Constants.SUBSCRIBE_KEY, true))) {
						registryList.add(url);
					}
				}
			}
		}
	}
	/**
	 * 返回的url为registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=dubbo-provider&application.version=1.0&dubbo=2.5.3&
	 * environment=product&organization=china&owner=cheng.xi&pid=17268&registry=zookeeper&timestamp=1488986530185
	 */
	return registryList;
}
```

加载注册中心URL的逻辑不算复杂，大概的步骤如下：

- 检查注册中心配置，如果不存在则抛异常；如果存在就填充各种Registry属性。
- 遍历多个注册中心，挨个进行URL的拼接。
- 将URL需要的参数映射都放到map中。
- 解析生成URL，并将需要返回的URL加入到List中返回。

最后返回的url是类似这种：`registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=dubbo-provider&application.version=1.0&dubbo=2.5.3&environment=product&organization=china&owner=cheng.xi&pid=17268&registry=zookeeper&timestamp=1488986530185`

## 一次一个协议导出服务

加载完注册中心URL之后，就是挨个去导出服务的过程了，具体的方法是doExportUrlsFor1Protocol：

```java

```



