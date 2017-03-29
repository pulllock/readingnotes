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
切面，持有一个Advice引用。子接口PointcutAdvisor持有Pointcut的引用。

```
public interface Advisor {
	boolean isPerInstance();
	//返回Advisor中持有的Advice
	Advice getAdvice();
}
```

## Advised
Advised接口支持多个Advisor或者Advice，加入了需要代理的源对象，需要代理的接口。实现类是AdvisedSupport。

## AdvisedSupport
实现了Advised接口，它可以动态添加，修改，删除通知和动态切换目标对象。

# 例子
使用接口的方式，配置文件：

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans>
    <!--业务处理类，也就是被代理的类-->
    <bean id="loginServiceImpl" class="me.cxis.spring.aop.LoginServiceImpl"/>

    <!--通知类-->
    <bean id="logBeforeLogin" class="me.cxis.spring.aop.LogBeforeLogin"/>

    <!--代理类-->
    <bean id="loginProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="proxyInterfaces">
            <value>me.cxis.spring.aop.LoginService</value>
        </property>
        <property name="interceptorNames">
            <list>
                <value>logBeforeLogin</value>
            </list>
        </property>
        <property name="target">
            <ref bean="loginServiceImpl"/>
        </property>
    </bean>
</beans>
```

LoginService：

```
package me.cxis.spring.aop;

public interface LoginService {
    String login(String userName);
}
```

LoginServiceImpl：

```
package me.cxis.spring.aop;

public class LoginServiceImpl implements LoginService {

    public String login(String userName){
        System.out.println("正在登录");
        return "success";
    }
}
```

LogBeforeLogin：

```
package me.cxis.spring.aop;

import org.springframework.aop.MethodBeforeAdvice;
import java.lang.reflect.Method;

public class LogBeforeLogin implements MethodBeforeAdvice {
    public void before(Method method, Object[] objects, Object o) throws Throwable {
        System.out.println("有人要登录了。。。");
    }
}
```

启动类Main：

```
package me.cxis.spring.aop;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Main {

    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:aop.xml");
        LoginService loginService = (LoginService) applicationContext.getBean("loginProxy");
        loginService.login("sdf");
    }
}
```


# 使用ProxyFactoryBean创建AOP代理
ProxyFactoryBean对Pointcut和Advice提供了完全的控制，还包括应用的顺序。ProxyFactoryBean的getObject方法会返回一个AOP代理，包装了目标对象。

Spring在初始化的过程中，createBean的时候，如果是FactoryBean的话，会调用` ((BeanFactoryAware)bean).setBeanFactory(this);`：

```
public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
    this.beanFactory = beanFactory;
    this.createAdvisorChain();
    if(this.singleton) {
        this.targetSource = this.freshTargetSource();
        //获取单例实例的时候
        this.getSingletonInstance();
        this.addListener(this);
    }

}
```

看下获取单例实例的方法：

```
private Object getSingletonInstance() {
    if(this.singletonInstance == null) {
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
	//Enhancer是CGLIB中的主要操作类
    Enhancer e = new Enhancer();
    try {
    	//从代理创建辅助类中获取在IoC容器中配置的目标对象
        Class rootClass = advised.getTargetSource().getTargetClass();
		//将目标对象本身做为enhancer的基类
        e.setSuperclass(rootClass);
        //将通知器中配置作为enhancer的方法过滤
        e.setCallbackFilter(new ProxyCallbackFilter(advised));

        e.setStrategy(new UndeclaredThrowableStrategy(UndeclaredThrowableException.class));
		//设置enhancer的接口
        e.setInterfaces(AopProxyUtils.completeProxiedInterfaces(advised));
		//设置enhancer的回调方法
        Callback[] callbacks = getCallbacks(rootClass);
        e.setCallbacks(callbacks);

        Class[] types = new Class[callbacks.length];

        for (int x = 0; x < types.length; x++) {
            types[x] = callbacks[x].getClass();
        }
        //设置enhancer的回调类型
        e.setCallbackTypes(types);
		//创建代理对象
        return e.create();
    }
    catch (CodeGenerationException ex) {。。。}
}
```
可以看到这里是使用cglib的方式去创建。当调用目标方法时，会去调用DynamicAdvisedInterceptor的intercept方法对目标对象进行处理：

```
public Object intercept(Object proxy, Method method, Object[] args,
        MethodProxy methodProxy) throws Throwable {

        MethodInvocation invocation = null;
        Object oldProxy = null;
        boolean setProxyContext = false;

        Class targetClass = null; //targetSource.getTargetClass();
        Object target = null;

        try {
            Object retVal = null;


            target = getTarget();
            if (target != null) {
                targetClass = target.getClass();
            }
			//如果通知器暴露了代理
            if (advised.exposeProxy) {
                //设置给定的代理对象为要被拦截的代理
                oldProxy = AopContext.setCurrentProxy(proxy);
                setProxyContext = true;
            }
			//获取AOP配置的通知
            List chain = advised.getAdvisorChainFactory()
                .getInterceptorsAndDynamicInterceptionAdvice(advised,
                    proxy, method, targetClass);

            //如果没有配置通知 
            if (chain.isEmpty()) {
                //直接调用目标对象的方法
                retVal = methodProxy.invoke(target, args);
            }
            else {
                //如果配置了通知
                //通过MethodInvocationImpl来启动配置的通知
                invocation = new MethodInvocationImpl(proxy, target,
                    method, args, targetClass, chain, methodProxy);
                retVal = invocation.proceed();
            }
			//获取目标对象对象方法的回调结果，如果有必要则封装为代理
            retVal = massageReturnTypeIfNecessary(proxy, target, retVal);
            return retVal;
        }
        finally {
            if (target != null) {
                releaseTarget(target);
            }

            if (setProxyContext) {
                // Restore old proxy
                AopContext.setCurrentProxy(oldProxy);
            }
        }
    }
```

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
	//通知的目标源
    TargetSource targetSource = advisedSupport.targetSource;
    Class targetClass = null;
    Object target = null;		

    try {
        
        if (method.getDeclaringClass() == Object.class && "equals".equals(method.getName())) {
            return equals(args[0]) ? Boolean.TRUE : Boolean.FALSE;
        }
        else if (Advised.class == method.getDeclaringClass()) {
            //如果AOP配置了通知，使用反射机制调用通知的同名方法
            return AopProxyUtils.invokeJoinpointUsingReflection(this.advisedSupport, method, args);
        }

        Object retVal = null;

        //目标对象
        target = targetSource.getTarget();
        if (target != null) {
            targetClass = target.getClass();
        }
		//如果当前通知暴露了代理，将当前代理使用currentProxy()方法变为可用代理
        if (this.advisedSupport.exposeProxy) {
            // Make invocation available if necessary
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        //获取目标对象方法配置的拦截器(通知器)链  
        List chain = this.advisedSupport.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                this.advisedSupport, proxy, method, targetClass);

        //没有配置任何通知
        if (chain.isEmpty()) {
            //没有配置通知，使用反射直接调用目标对象的方法，并获取方法返回值
            retVal = AopProxyUtils.invokeJoinpointUsingReflection(target, method, args);
        }
        else {//配置了通知
        	//为目标对象创建方法回调对象，需要在调用通知之后才调用目标对象的方法
            invocation = new ReflectiveMethodInvocation(proxy, target,
                                method, args, targetClass, chain);

            //调用通知链，沿着通知器链调用所有配置的通知
            retVal = invocation.proceed();
        }

        //如果方法有返回值，则将代理对象最为方法返回
        if (retVal != null && retVal == target) {
            retVal = proxy;
        }
        return retVal;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            //释放目标对象
            targetSource.releaseTarget(target);
        }

        if (setProxyContext) {
            //存储代理对象
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```
