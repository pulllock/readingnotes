IOC容器初始化过程包括了读取XML资源，载入解析，注册。而对于Bean的依赖注入并不包含在IOC容器的初始化过程中。完成初始化IOC容器之后，如果Bean没有配置延迟加载属性或者bean是单例的，那么就会开始对bean进行实例化。其他的bean都是延迟加载，需要在第一次调用getBean的时候被创建。

对于bean的依赖注入是在refresh()方法的：``finishBeanFactoryInitialization(beanFactory);``这个地方开始的：

```
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
  ......
	//这里实例化非延迟加载的单例bean
	beanFactory.preInstantiateSingletons();
}
```
preInstantiateSingletons方法在DefaultListableBeanFactory中：

```
public void preInstantiateSingletons() throws BeansException {
	List<String> beanNames;
	synchronized (this.beanDefinitionMap) {
		beanNames = new ArrayList<String>(this.beanDefinitionNames);
	}
	for (String beanName : beanNames) {
		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
    //对非抽象、单例的和非延迟加载的对象进行实例化
		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			if (isFactoryBean(beanName)) {
				final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
				boolean isEagerInit;
				if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
					isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
						public Boolean run() {
							return ((SmartFactoryBean<?>) factory).isEagerInit();
						}
					}, getAccessControlContext());
				}
				else {
					isEagerInit = (factory instanceof SmartFactoryBean &&
							((SmartFactoryBean<?>) factory).isEagerInit());
				}
				if (isEagerInit) {
					getBean(beanName);
				}
			}
			else {
				getBean(beanName);
			}
		}
	}
}
```

无论预先创建还是延迟加载都是调用getBean方法实现的，getBean直接调用doGetBean方法：

```
protected <T> T doGetBean(
		final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
		throws BeansException {
  //获取beanName
	final String beanName = transformedBeanName(name);
	Object bean;

	//早期检查，从缓存中获取单例实例
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
		//从缓存中获取实例
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}

	else {
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}

		BeanFactory parentBeanFactory = getParentBeanFactory();
    //父容器存在，并且当前容器不包含该bean，就去父容器中查找
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			String nameToLookup = originalBeanName(name);
			if (args != null) {
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else {
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
		}

		if (!typeCheckOnly) {
			markBeanAsCreated(beanName);
		}

		final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
		checkMergedBeanDefinition(mbd, beanName, args);

		// 确保当前bean所依赖的bean先实例化
		String[] dependsOn = mbd.getDependsOn();
		if (dependsOn != null) {
			for (String dependsOnBean : dependsOn) {
				getBean(dependsOnBean);
				registerDependentBean(dependsOnBean, beanName);
			}
		}

		// 创建bean的实例，单例bean，通过getSingleton方法来创建，创建完后会将单例对象缓存。
		if (mbd.isSingleton()) {
			sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
				public Object getObject() throws BeansException {
					try {
						return createBean(beanName, mbd, args);
					}
				}
			});
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
		}

		else if (mbd.isPrototype()) {//非单例的bean的实例的创建
			// It's a prototype -> create a new instance.
			Object prototypeInstance = null;
			try {
				beforePrototypeCreation(beanName);
				prototypeInstance = createBean(beanName, mbd, args);
			}
			finally {
				afterPrototypeCreation(beanName);
			}
			bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
		}

		else {//其他bean的实例化
			String scopeName = mbd.getScope();
			final Scope scope = this.scopes.get(scopeName);
			if (scope == null) {
				throw new IllegalStateException("No Scope registered for scope '" + scopeName + "'");
			}
			try {
				Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
					public Object getObject() throws BeansException {
						beforePrototypeCreation(beanName);
						try {
							return createBean(beanName, mbd, args);
						}
						finally {
							afterPrototypeCreation(beanName);
						}
					}
				});
				bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
			}
		}
	}

	// Check if required type matches the type of the actual bean instance.
	if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
		try {
			return getTypeConverter().convertIfNecessary(bean, requiredType);
		}
	}
	return (T) bean;
}
```

单例对象和非单例对象的创建过程是一样的，都是调用父类AbstractAutowireCapableBeanFactory的createBean方法。单例对象创建的时候会调用getSingleton方法，getSingleton方法中也会间接调用createBean方法：

```
public Object getSingleton(String beanName, ObjectFactory singletonFactory) {
	synchronized (this.singletonObjects) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null) {
			beforeSingletonCreation(beanName);
			boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
			if (recordSuppressedExceptions) {
				this.suppressedExceptions = new LinkedHashSet<Exception>();
			}
			try {
        //创建bean
				singletonObject = singletonFactory.getObject();
			}
			finally {
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = null;
				}
				afterSingletonCreation(beanName);
			}
      //添加到缓存中
			addSingleton(beanName, singletonObject);
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}
}
```

看下createBean方法：

```
protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
			throws BeanCreationException {
	// Make sure bean class is actually resolved at this point.
	resolveBeanClass(mbd, beanName);

	// Prepare method overrides.
	try {
		mbd.prepareMethodOverrides();
	}
	try {
		// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
		Object bean = resolveBeforeInstantiation(beanName, mbd);
		if (bean != null) {
			return bean;
		}
	}

	Object beanInstance = doCreateBean(beanName, mbd, args);
	return beanInstance;
}
```
createBean方法会调用doCreateBean方法：

```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
	//创建一个BeanWrapper对象，用于存放实例化对象
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) {
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
  //创建bean实例，会使用反射，BeanUtils 的 instantiateClass 方法，通过反射创建对象
	if (instanceWrapper == null) {
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
	Class beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);

	// Allow post-processors to modify the merged bean definition.
	synchronized (mbd.postProcessingLock) {
		if (!mbd.postProcessed) {
			applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
			mbd.postProcessed = true;
		}
	}

	// Eagerly cache singletons to be able to resolve circular references
	// even when triggered by lifecycle interfaces like BeanFactoryAware.
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
			isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		addSingletonFactory(beanName, new ObjectFactory() {
			public Object getObject() throws BeansException {
				return getEarlyBeanReference(beanName, mbd, bean);
			}
		});
	}

	// Initialize the bean instance.
	Object exposedObject = bean;
	try {
    //初始化bean的实例，属性注入，根据注入方式进行注入
		populateBean(beanName, mbd, instanceWrapper);
		if (exposedObject != null) {
      //先判断是否实现了BeanNameAware 、 BeanClassLoaderAware 等接口，如果实现了，进行默认的注入。
      //接着调用applyBeanPostProcessorsBeforeInitialization
      //然后如果实现 InitializingBean 接口，调用 afterPropertySet 方法。
      //最后调用applyBeanPostProcessorsAfterInitialization
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
	}

	if (earlySingletonExposure) {
		Object earlySingletonReference = getSingleton(beanName, false);
		if (earlySingletonReference != null) {
			if (exposedObject == bean) {
				exposedObject = earlySingletonReference;
			}
			else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
				String[] dependentBeans = getDependentBeans(beanName);
				Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
				for (String dependentBean : dependentBeans) {
					if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
						actualDependentBeans.add(dependentBean);
					}
				}
			}
		}
	}
	// Register bean as disposable.
	try {
		registerDisposableBeanIfNecessary(beanName, bean, mbd);
	}

	return exposedObject;
}
```

bean的依赖注入过程就已经完成了。
