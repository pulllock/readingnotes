有关AOP相关概念以及Spring AOP相关概念不再重复。

# Pointcut
## Pointcut接口
切入点，定义了哪些连接点需要被织入横切逻辑。

```
public interface Pointcut {
	//类过滤器，可以知道哪些类需要拦截
	ClassFilter getClassFilter();
	//方法匹配器，可以知道哪些方法需要拦截
	MethodMatcher getMethodMatcher();
	
	// could add getFieldMatcher() without breaking most existing code
	Pointcut TRUE = TruePointcut.INSTANCE; 

}
```
## ClassFilter接口

```
public interface ClassFilter {
	
	//判断给定的类是不是要拦截
	boolean matches(Class clazz);

	ClassFilter TRUE = TrueClassFilter.INSTANCE;

}
```


## MethodMatcher接口

```
public interface MethodMatcher {
	
	/ 静态方法匹配
	boolean matches(Method m, Class targetClass);
	
	//是否是运行时动态匹配
	boolean isRuntime();
	
	//运行是动态匹配
	boolean matches(Method m, Class targetClass, Object[] args);
	
	MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;

}
```

# Advice
Advice是通知，代表的是具体的横切逻辑的定义。Advice不是Spring定义的，在`org.aopalliance.aop.Advice`中，Spring定义的Advice大概有：

- BeforeAdvice，仅支持方法，在连接点前执行的横切逻辑。
- MethodBeforeAdvice，继承自BeforeAdvice。
- ThrowsAdvice，抛异常的时候，执行
- AfterReturningAdvice，仅支持方法，只在正常的方法返回之后才会执行，不能更改方法的返回值

## 关于AroundAdvice
Spring中并没有定义AroundAdvice，而是使用`org.aopalliance.intercept.MethodInterceptor`接口，该接口只有一个invoke方法，我们只需要实现此接口并实现方法就可。

## 关于Introduction advice
Introduction advice需要一个IntroductionAdvisor和一个IntroductionInterceptor配合。不常用，暂先不解释。

# Advisor
切面，包含一个Pointcut和一个Advice。

```
public interface Advisor {
	boolean isPerInstance();
	//返回Advisor中持有的Advice
	Advice getAdvice();
}
```

# 使用ProxyFactoryBean创建AOP代理
ProxyFactoryBean对Pointcut和Advice提供了完全的控制，还包括应用的顺序。ProxyFactoryBean的getObject方法会返回一个AOP代理，包装了目标对象。首先从getObject开始切入：

```
//当客户端获取bean的时候，会调用，返回一个代理
public Object getObject() throws BeansException {
	//判断是单例的还是prototype的
    return (this.singleton) ? getSingletonInstance() : newPrototypeInstance();
}
```

获取单例AOP代理的实例getSingletonInstance()：

```
private Object getSingletonInstance() {
	//单例的会被缓存，所以先从缓存中查询
    if (this.singletonInstance == null) {
        this.singletonInstance = this.createAopProxy().getProxy();
    }
    return this.singletonInstance;
}
```

createAopProxy方法在AdvisedSupport类中：

```
//创建新的AOP代理
protected synchronized AopProxy createAopProxy() {
    if (!this.isActive) {
        activate();
    }
    return getAopProxyFactory().createAopProxy(this);
}
```
getAopProxyFactory获取AOP代理工厂，代理工厂是DefaultAopProxyFactory类，获取到工厂之后，然后createAopProxy，创建AOP代理，下面去DefaultAopProxyFactory中看看：

```
public AopProxy createAopProxy(AdvisedSupport advisedSupport) throws AopConfigException {
	//optimize指的是代理对象是否采取优化，true就是使用cglib
    boolean useCglib = advisedSupport.getOptimize() || advisedSupport.getProxyTargetClass() || advisedSupport.getProxiedInterfaces().length == 0;
    //使用Cglib生成AOP代理
    if (useCglib) {
        return CglibProxyFactory.createCglibProxy(advisedSupport);
    }
    else {//使用JDK动态代理来生成AOP代理
        // Depends on whether we have expose proxy or frozen or static ts
        return new JdkDynamicAopProxy(advisedSupport);
    }
}
```

## CGLIB创建代理
首先看下使用CGLIB创建代理`CglibProxyFactory.createCglibProxy(advisedSupport);`，其实就是直接返回一个Cglib2AopProxy实例：

```
protected Cglib2AopProxy(AdvisedSupport config) throws AopConfigException {
    if (config == null) {
        throw new AopConfigException("Cannot create AopProxy with null ProxyConfig");
    }
    if (config.getAdvisors().length == 0 &&
        config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
        throw new AopConfigException("Cannot create AopProxy with no advisors and no target source");
    }
    this.advised = config;
    if (this.advised.getTargetSource().getTargetClass() == null) {
        throw new AopConfigException("Either an interface or a target is required for proxy creation");
    }
}
```
获取到代理的方法是getProxy方法，查看一下：

```
public Object getProxy() {
    //使用当前线程的类加载器
    return getProxy(Thread.currentThread().getContextClassLoader());
}
```
继续：

```
public Object getProxy(ClassLoader cl) {
    Enhancer e = new Enhancer();
    try {
        Class rootClass = advised.getTargetSource().getTargetClass();

        e.setSuperclass(rootClass);
        e.setCallbackFilter(new ProxyCallbackFilter(advised));

        e.setStrategy(new UndeclaredThrowableStrategy(UndeclaredThrowableException.class));


        e.setInterfaces(AopProxyUtils.completeProxiedInterfaces(advised));

        Callback[] callbacks = getCallbacks(rootClass);

        e.setCallbacks(callbacks);

        Class[] types = new Class[callbacks.length];

        for (int x = 0; x < types.length; x++) {
            types[x] = callbacks[x].getClass();
        }
        e.setCallbackTypes(types);

        return e.create();
    }
    catch (CodeGenerationException ex) {。。。}
}
```
可以看到这里是使用cglib的方式去创建，不太熟悉，暂先略过。

## 使用JDK动态代理生成AOP代理
也是直接返回一个实例`new JdkDynamicAopProxy(advisedSupport);`。获取代理的方法同样是getProxy方法：

```
public Object getProxy() {
		//使用当前线程的类加载器
        return getProxy(Thread.currentThread().getContextClassLoader());
	}
```
继续往下：

```
public Object getProxy(ClassLoader cl) {
    //根据advised中的配置信息，获取需要代理的接口
    Class[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advisedSupport);
    //使用JDK的动态代理来生成Proxy实例
    return Proxy.newProxyInstance(cl, proxiedInterfaces, this);
}
```
这样就会得到代理对象了，然后业务中调用获取到的bean的方法的时候，对其方法的调用，就会调用代理对象的invoke方法，看下invoke方法：

```
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	//具体方法调用
    MethodInvocation invocation = null;
    //原来的代理
    Object oldProxy = null;
    boolean setProxyContext = false;

    TargetSource targetSource = advisedSupport.targetSource;
    Class targetClass = null;
    Object target = null;		

    try {
        
        if (method.getDeclaringClass() == Object.class && "equals".equals(method.getName())) {
            return equals(args[0]) ? Boolean.TRUE : Boolean.FALSE;
        }
        else if (Advised.class == method.getDeclaringClass()) {
            //目标对象方法的调用
            return AopProxyUtils.invokeJoinpointUsingReflection(this.advisedSupport, method, args);
        }

        Object retVal = null;

        // May be null. Get as late as possible to minimize the time we "own" the target,
        // in case it comes from a pool.
        target = targetSource.getTarget();
        if (target != null) {
            targetClass = target.getClass();
        }

        if (this.advisedSupport.exposeProxy) {
            // Make invocation available if necessary
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        // Get the interception chain for this method
        List chain = this.advisedSupport.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                this.advisedSupport, proxy, method, targetClass);

        // Check whether we have any advice. If we don't, we can fallback on
        // direct reflective invocation of the target, and avoid creating a MethodInvocation
        if (chain.isEmpty()) {
            // We can skip creating a MethodInvocation: just invoke the target directly
            // Note that the final invoker must be an InvokerInterceptor so we know it does
            // nothing but a reflective operation on the target, and no hot swapping or fancy proxying
            retVal = AopProxyUtils.invokeJoinpointUsingReflection(target, method, args);
        }
        else {
            // We need to create a method invocation...
            //invocation = advised.getMethodInvocationFactory().getMethodInvocation(proxy, method, targetClass, target, args, chain, advised);

            invocation = new ReflectiveMethodInvocation(proxy, target,
                                method, args, targetClass, chain);

            // Proceed to the joinpoint through the interceptor chain
            retVal = invocation.proceed();
        }

        // Massage return value if necessary
        if (retVal != null && retVal == target) {
            // Special case: it returned "this"
            // Note that we can't help if the target sets
            // a reference to itself in another returned object
            retVal = proxy;
        }
        return retVal;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            // Must have come from TargetSource
            targetSource.releaseTarget(target);
        }

        if (setProxyContext) {
            // Restore old proxy
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```

# Spring在bean初始化过程中对AOP的处理
在AbstractAutowireCapableBeanFactory的createBean方法中，会调用applyBeanPostProcessorsAfterInitialization(bean, beanName);方法：

```
public Object applyBeanPostProcessorsAfterInitialization(Object bean, String name) throws BeansException {
    Object result = bean;
    for (Iterator it = getBeanPostProcessors().iterator(); it.hasNext();) {
        BeanPostProcessor beanProcessor = (BeanPostProcessor) it.next();
        //这里进入AbstractAutoProxyCreator的postProcessAfterInitialization方法
        result = beanProcessor.postProcessAfterInitialization(result, name);
        if (result == null) {
            throw new BeanCreationException( );
        }
    }
    return result;
}
```
AbstractAutoProxyCreator的postProcessAfterInitialization方法：

```
//使用配置的interceptor来创建一个代理
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    // Check for special cases. We don't want to try to autoproxy a part of the autoproxying
    // infrastructure, lest we get a stack overflow.
    if (isInfrastructureClass(bean, beanName) || shouldSkip(bean, beanName)) {
        logger.debug("Did not attempt to autoproxy infrastructure class [" + bean.getClass().getName() + "]");
        return bean;
    }

    TargetSource targetSource = getCustomTargetSource(bean, beanName);

    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean, beanName, targetSource);

    // proxy if we have advice or if a TargetSourceCreator wants to do some
    // fancy stuff such as pooling
    if (specificInterceptors != DO_NOT_PROXY || targetSource != null) {

        if (targetSource == null) {
            // use default of simple, default target source
            targetSource = new SingletonTargetSource(bean);
        }

        // handle prototypes correctly
        Advisor[] commonInterceptors = resolveInterceptorNames();

        List allInterceptors = new ArrayList();
        if (specificInterceptors != null) {
            allInterceptors.addAll(Arrays.asList(specificInterceptors));
            if (commonInterceptors != null) {
                if (this.applyCommonInterceptorsFirst) {
                    allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
                }
                else {
                    allInterceptors.addAll(Arrays.asList(commonInterceptors));
                }
            }
        }

        ProxyFactory proxyFactory = new ProxyFactory();
        // copy our properties (proxyTargetClass) inherited from ProxyConfig
        proxyFactory.copyFrom(this);

        if (!getProxyTargetClass()) {
            // Must allow for introductions; can't just set interfaces to
            // the target's interfaces only.
            Class[] targetsInterfaces = AopUtils.getAllInterfaces(bean);
            for (int i = 0; i < targetsInterfaces.length; i++) {
                proxyFactory.addInterface(targetsInterfaces[i]);
            }
        }

        for (Iterator it = allInterceptors.iterator(); it.hasNext();) {
            Advisor advisor = this.advisorAdapterRegistry.wrap(it.next());
            proxyFactory.addAdvisor(advisor);
        }
        proxyFactory.setTargetSource(targetSource);
        customizeProxyFactory(bean, proxyFactory);
		//获取并返回代理对象，这就跟上面的一样了，不再说明
        return proxyFactory.getProxy();
    }
    else {
        return bean;
    }
}
```
