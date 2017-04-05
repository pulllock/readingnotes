初始化从refresh()方法开始

# prepareRefresh()
初始化准备工作，设置启动时间，active标志，初始化占位符属性资源等，对系统变量和环境变量进行验证准备。

## initPropertySources()
初始化占位符属性资源等。

## validateRequiredProperties()
校验被标记为required的属性。

# obtainFreshBeanFactory()
容器的初始化，包括读取xml进行Resource的定位，载入解析，注册等。

## refreshBeanFactory()
刷新BeanFactory。执行真正的配置加载。该方法会在其他初始化工作开始之前被调用。

需要先销毁bean，就是执行各种map的clear方法之类的。然后需要关闭之前的BeanFactory，就是置为null。

接下来会创建BeanFactory，设置序列化id，自定义BeanFactory，加载BeanDefinitions
。

### createBeanFactory()
为当前上下文创建一个内部Bean工厂，会先获取父工厂，然后返回一个DefaultListableBeanFactory实例，在返回DefaultListableBeanFactory实例过程中会忽略给定接口的自动装配功能，这里忽略三个接口BeanNameAware，BeanFactoryAware，BeanClassLoaderAware。
### customizeBeanFactory()
### loadBeanDefinitions()
## getBeanFactory()
获取BeanFactory。