Spring中常见的Aware接口有BeanFactoryAware，BeanNameAware，ApplicationContextAware，ResourceLoaderAware，BeanClassLoaderAware，ServletContextAware。实现这些接口的Bean在初始化之后可以方便的从上下文中获取当前的运行环境。比如：Bean实现了BeanFactoryAware接口，在初始化之后，Spring容器会注入BeanFactory实例。

Bean取得了当前运行环境之后，就可以获取相关的资源，使用相关的机制等。比如ApplicationContext提供了publishEvent方法，就可以调用。

BeanNameAware接口：某个Bean需要访问配置文件中bean本身的id属性，可以实现该接口。在依赖关系确定之后，初始化方法之前，提供回调自身的能力，从而获得bean本身的id属性。该接口提供了setBeanName方法，name参数是bean的id属性，调用此方法可以让bean获得自身的id属性。

BeanFactoryAware接口：实现该接口后，Bean可以通过BeanFactory来访问Spring的容器。这样容易导致代码和Spring的api偶合在一起。

ApplicationContextAware接口：实现该接口，在Bean被实例化之后，将会被注入ApplicationContext实例。这样容易导致代码和Spring的api偶合在一起。


容器对于Aware接口的管理实现，主要代码在AbstractAutowireCapableBeanFactory的initializeBean方法中：

```
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
	if (System.getSecurityManager() != null) {
		AccessController.doPrivileged(new PrivilegedAction<Object>() {
			public Object run() {
				invokeAwareMethods(beanName, bean);
				return null;
			}
		}, getAccessControlContext());
	}
	else {
		//对于Aware接口的处理在这里
		invokeAwareMethods(beanName, bean);
	}

	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
		//开始Bean初始化之前的处理
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}

	try {
		//Bean初始化
		invokeInitMethods(beanName, wrappedBean, mbd);
	}
	

	if (mbd == null || !mbd.isSynthetic()) {
		//Bean初始化后的处理
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}
	return wrappedBean;
}
```

对于Aware接口的处理在invokeAwareMethods方法里面：

```
private void invokeAwareMethods(final String beanName, final Object bean) {
	//判断对象的接口类型
	//对BeanNameAware，BeanClassLoaderAware，BeanFactoryAware三种类型的接口进行处理。
	if (bean instanceof Aware) {
		if (bean instanceof BeanNameAware) {
			((BeanNameAware) bean).setBeanName(beanName);
		}
		if (bean instanceof BeanClassLoaderAware) {
			((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
		}
		if (bean instanceof BeanFactoryAware) {
			((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
		}
	}
}
```

Aware接口的各种处理是在属性设置完成之后，Bean初始化之前完成的。