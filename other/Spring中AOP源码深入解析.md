有关AOP相关概念以及Spring AOP相关概念和Spring AOP的使用不再重复。关于AOP在Spring中的地位，不用说相信我们都知道，也都会用，但是对于更深入的东西，还未接触过，这里就对Spring AOP的相关源码进行说明一下，看看到底Spring中AOP是怎么实现的。

有关AOP的概念和Spring AOP相关配置，请参考其他两篇文章：[AOP概念，原理，应用介绍](http://cxis.me/2017/04/12/AOP%E6%A6%82%E5%BF%B5%EF%BC%8C%E5%8E%9F%E7%90%86%EF%BC%8C%E5%BA%94%E7%94%A8%E4%BB%8B%E7%BB%8D/) 和 [Spring中AOP的配置从1.0到5.0的演进](http://cxis.me/2017/04/10/Spring%E4%B8%ADAOP%E7%9A%84%E9%85%8D%E7%BD%AE%E4%BB%8E1.0%E5%88%B05.0%E7%9A%84%E6%BC%94%E8%BF%9B/)

另外，本文使用的源码是Spring1.1.1版本的，之所以使用这么老的版本，是觉得相对来说简单一些，并且无关的东西更少，这样更容易去理解。对于后续版本新增功能可以在此基础上进行对比，理解的效果会更好。

# 示例程序
首先我们还是先使用一个实例来看一下怎么使用，再从实例中一步一步跟进到源码中。

先定义业务接口和实现：

LoginService：

```
package me.cxis.spring.aop;

/**
 * Created by cheng.xi on 2017-03-29 12:02.
 */
public interface LoginService {
    String login(String userName);
}
```

LoginServiceImpl：

```
package me.cxis.spring.aop;

/**
 * Created by cheng.xi on 2017-03-29 10:36.
 */
public class LoginServiceImpl implements LoginService {

    public String login(String userName){
        System.out.println("正在登录");
        return "success";
    }
}
```

接着是三个通知类：

```
//这里只是在登录方法调用之前打印一句话
public class LogBeforeLogin implements MethodBeforeAdvice {
    public void before(Method method, Object[] objects, Object o) throws Throwable {
        System.out.println("有人要登录了。。。");
    }
}

package me.cxis.spring.aop.proxyfactory;

import org.springframework.aop.AfterReturningAdvice;
import org.springframework.aop.MethodBeforeAdvice;

import java.lang.reflect.Method;

/**
 * Created by cheng.xi on 2017-03-29 10:56.
 */
public class LogAfterLogin implements AfterReturningAdvice {

    public void afterReturning(Object o, Method method, Object[] objects, Object o1) throws Throwable {
        System.out.println("有人已经登录了。。。");
    }
}

package me.cxis.spring.aop.proxyfactory;

import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;

/**
 * Created by cheng.xi on 2017-03-30 23:36.
 */
public class LogAroundLogin implements MethodInterceptor {
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("有人要登录。。。");
        Object result = invocation.proceed();
        System.out.println("登录完了");
        return result;
    }
}

```

测试方法：

```
package me.cxis.spring.aop.proxyfactory;

import org.springframework.aop.framework.ProxyFactory;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * Created by cheng.xi on 2017-03-29 10:34.
 */
public class Main {


    public static void main(String[] args) {
        ProxyFactory proxyFactory = new ProxyFactory();//创建代理工厂
        proxyFactory.setTarget(new LoginServiceImpl());//设置目标对象
        proxyFactory.addAdvice(new LogBeforeLogin());//前置增强
        proxyFactory.addAdvice(new LogAfterLogin());//后置增强
        //proxyFactory.addAdvice(new LogAroundLogin());//环绕增强

        LoginService loginService = (LoginService) proxyFactory.getProxy();//从代理工厂中获取代理
        loginService.login("x");
    }
}

```

关于实例中要说明的：我们看到在使用的时候，直接获取的是一个代理，不是要使用的实现类，这也很好懂，之前文章都说过AOP其实就是代理模式，在编译期或者运行期，给我们原来的代码增加一些功能，变成一个代理。当我们调用的时候，实际就是调用的代理类。

# 源码解析
对于源码的解析，我们这里使用的是代码的方式，没有选择xml配置文件的方式。关于xml配置的方式，后面再讲解。

## 创建AOP代理
首先我们要明白，Spring中实现AOP，就是生成一个代理，然后在使用的时候调用代理。

### 创建代理工厂
代码中首先创建一个代理工厂实例`ProxyFactory proxyFactory = new ProxyFactory();`代理工厂的作用就是使用编程的方式创建AOP代理。ProxyFactory继承自AdvisedSupport，AdvicedSupport是AOP代理的配置管理器。

### 设置目标对象
然后是设置要代理的目标对象`proxyFactory.setTarget(new LoginServiceImpl());`，看下setTarget方法：

```
public void setTarget(Object target) {
	//先根据给定的目标实现类，创建一个单例的TargetSource
    //然后设置TargetSource
    setTargetSource(new SingletonTargetSource(target));
}
```

#### TargetSource
TargetSource用来获取当前的Target，也就是TargetSource中会保存着我们的的实现类。

```
public interface TargetSource {

	//返回目标类的类型
	Class getTargetClass();
	
	//查看TargetSource是否是static的
    //静态的TargetSource每次都返回同一个Target
	boolean isStatic();
	
	//获取目标类的实例
	Object getTarget() throws Exception;
	
	//释放目标类
	void releaseTarget(Object target) throws Exception;

}
```

#### SingletonTargetSource
TargetSource的默认实现，是一个单例的TargetSource，isStatic方法直接返回true。

```
public final class SingletonTargetSource implements TargetSource, Serializable {

	//用来保存目标类	
	private final Object target;
	//构造方法
	public SingletonTargetSource(Object target) {
		this.target = target;
	}
	//直接返回目标类的类型
	public Class getTargetClass() {
		return target.getClass();
	}
	//返回目标类
	public Object getTarget() {
		return this.target;
	}
	//释放目标类，这里啥也没做
	public void releaseTarget(Object o) {
		// Nothing to do
	}
	//直接返回true
	public boolean isStatic() {
		return true;
	}

	//equals方法
	public boolean equals(Object other) {
    	//相等，返回true
		if (this == other) {
			return true;
		}
        //不是SingletonTargetSource类型的返回false
		if (!(other instanceof SingletonTargetSource)) {
			return false;
		}
		SingletonTargetSource otherTargetSource = (SingletonTargetSource) other;
        //判断目标类是否相等
		return ObjectUtils.nullSafeEquals(this.target, otherTargetSource.target);
	}
	
	//toString方法
	public String toString() {
		return "SingletonTargetSource: target=(" + target + ")";
	}
}
```

上面是有关TargetSource和SingletonTargetSource的说明，接着往下一步就是设置目标类setTargetSource方法，在AdvisedSupport类中：

```
public void setTargetSource(TargetSource targetSource) {
    if (isActive() && getOptimize()) {
        throw new AopConfigException("Can't change target with an optimized CGLIB proxy: it has its own target");
    }
    //么有做什么处理，只是将我们构建的TargetSource缓存起来
    this.targetSource = targetSource;
}
```
### 添加通知
上面设置了要代理的目标类之后，接着是添加通知，也就是添加增强类，`proxyFactory.addAdvice()`方法是添加增强类的方法。我们在例子中是这么使用的：

```
proxyFactory.addAdvice(new LogBeforeLogin());//前置增强
proxyFactory.addAdvice(new LogAfterLogin());//后置增强
//proxyFactory.addAdvice(new LogAroundLogin());//环绕增强
```
addAdvice方法的参数是一个Advice类型的类，也就是通知或者叫增强，可以去我们的增强类中查看，我们都继承了各种Advice，比如`MethodBeforeAdvice`，`AfterReturningAdvice`，`MethodInterceptor`，这里先讲一下有关通知Advice的代码，然后再继续说明addAdvice方法。

#### Advice接口
Advice不属于Spring，是AOP联盟定义的接口。Advice接口并没有定义任何方法，是一个空的接口，用来做标记，实现了此接口的的类是一个通知类。Advice有几个子接口：

- BeforeAdvice，前置增强，意思是在我们的目标类之前调用的增强。这个接口也没有定义任何方法。
- AfterReturningAdvice，方法正常返回前的增强，该增强可以看到方法的返回值，但是不能更改返回值，该接口有一个方法afterReturning
- ThrowsAdvice，抛出异常时候的增强，也是一个标志接口，没有定义任何方法。
- Interceptor，拦截器，也没有定义任何方法，表示一个通用的拦截器。不属于Spring，是AOP联盟定义的接口
- DynamicIntroductionAdvice，动态引介增强，有一个方法implementsInterface。

##### MethodBeforeAdvice
MethodBeforeAdvice接口，是BeforeAdvice的子接口，表示在方法前调用的增强，方法前置增强不能阻止方法的调用，但是能抛异常来使目标方法不继续执行。

```
public interface MethodBeforeAdvice extends BeforeAdvice {
	
	//在给定的方法调用前，调用该方法
    //参数method是被代理的方法
    //参数args是被代理方法的参数
    //参数target是方法调用的目标，可能为null
	void before(Method m, Object[] args, Object target) throws Throwable;
}
```

##### MethodInterceptor
MethodInterceptor不属于Spring，是AOP联盟定义的接口，是Interceptor的子接口，我们通常叫做环绕增强。

```
public interface MethodInterceptor extends Interceptor {
	
    //在目标方法调用前后做一些事情
    //返回的是invocation.proceed()方法的返回值
    Object invoke(MethodInvocation invocation) throws Throwable;
}
```
参数MethodInvocation是一个方法调用的连接点，接下来先看看MethodInvocation相关的代码。

##### Joinpoint连接点
JointPoint接口，是一个通用的运行时连接点，运行时连接点是在一个静态连接点发生的事件。

```
public interface Joinpoint {

   //开始调用拦截器链中的下一个拦截器
   Object proceed() throws Throwable;

   //
   Object getThis();

   //
   AccessibleObject getStaticPart();   

}
```

##### Invocation接口
Invocation接口是Joinpoint的子接口，表示程序的调用，一个Invocation就是一个连接点，可以被拦截器拦截。

```
public interface Invocation extends Joinpoint {
   
   //获取参数
   Object[] getArguments();

}
```

##### MethodInvocation接口
MethodInvocation接口是Invocation的子接口，用来描述一个方法的调用。

```
public interface MethodInvocation extends Invocation
{

    //获取被调用的方法
    Method getMethod();

}
```
另外还有一个ConstructorInvocation接口，也是Invocation的子接口，描述的是构造器的调用。

上面介绍完了Advice的相关定义，接着看往代理工厂中添加增强的addAdvice方法，addAdvice方法在AdvisedSupport类中：

```
public void addAdvice(Advice advice) throws AopConfigException {
	//advisors是Advice列表，是一个LinkedList
    //如果被添加进来的是一个Interceptor，会先被包装成一个Advice
    //添加之前现获取advisor的大小，当做添加的Advice的位置
    int pos = (this.advisors != null) ? this.advisors.size() : 0;
    //添加Advice
    addAdvice(pos, advice);
}
```
接着看addAdvice(pos, advice)方法：

```
public void addAdvice(int pos, Advice advice) throws AopConfigException {
	//只能处理实现了AOP联盟的接口的拦截器
    if (advice instanceof Interceptor && !(advice instanceof MethodInterceptor)) {
        throw new AopConfigException(getClass().getName() + " only handles AOP Alliance MethodInterceptors");
    }
	//IntroductionInfo接口类型，表示引介信息
    if (advice instanceof IntroductionInfo) {
    	//不需要IntroductionAdvisor
        addAdvisor(pos, new DefaultIntroductionAdvisor(advice, (IntroductionInfo) advice));
    }
    //动态引介增强的处理
    else if (advice instanceof DynamicIntroductionAdvice) {
        //需要IntroductionAdvisor
        throw new AopConfigException("DynamicIntroductionAdvice may only be added as part of IntroductionAdvisor");
    }
    else {
    	//添加增强器，需要先把我们的增强包装成增强器，然后添加
        addAdvisor(pos, new DefaultPointcutAdvisor(advice));
    }
}
```
我们看到添加增强的时候，实际调用添加增强器这个方法，首先需要把我们的Advice包装成一个PointCutAdvisor，然后在添加增强器。这里先了解一下有关PointCutAdvisor的相关信息。

#### Advisor接口
Advisor，增强器，它持有一个增强Advice，还持有一个过滤器，来决定Advice可以用在哪里。

```
public interface Advisor {
	
	//判断Advice是不是每个实例中都有
	boolean isPerInstance();
	
	//返回持有的Advice
	Advice getAdvice();

}
```

#### PointcutAdvisor
是一个持有Pointcut切点的增强器，PointcutAdvisor现在就会持有一个Advice和一个Pointcut。

```
public interface PointcutAdvisor extends Advisor {

	//获取Pointcut
	Pointcut getPointcut();

}
```

#### Pointcut接口
切入点，定义了哪些连接点需要被织入横切逻辑。可以

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
#### ClassFilter接口

```
public interface ClassFilter {
	
	//判断给定的类是不是要拦截
	boolean matches(Class clazz);

	ClassFilter TRUE = TrueClassFilter.INSTANCE;

}
```


#### MethodMatcher接口

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

看完相关的定义之后，接着看方法new DefaultPointcutAdvisor(advice)，将Advice包装成一个DefaultPointcutAdvisor。其实就是将advice和默认的Pointcut包装进DefaultPointcutAdvisor。

DefaultPointcutAdvisor是Advisor的最常用的一个实现，可以使用任意类型的Pointcut和Advice，但是不能使用Introduction。

构造完成了DefaultPointcutAdvisor只有，接着就是添加增强器方法addAdvisor：

```
public void addAdvisor(int pos, Advisor advisor) throws AopConfigException {
	//引介增强器处理
    if (advisor instanceof IntroductionAdvisor) {
        addAdvisor(pos, (IntroductionAdvisor) advisor);
    }
    else {
    	//其他的增强器处理
        addAdvisorInternal(pos, advisor);
    }
}
```

首先看下非引介增强器的添加方法addAdvisorInternal：

```
private void addAdvisorInternal(int pos, Advisor advice) throws AopConfigException {
    if (isFrozen()) {
        throw new AopConfigException("Cannot add advisor: config is frozen");
    }
    //把Advice添加到LinkedList中指定位置
    this.advisors.add(pos, advice);
    //同时更新一下Advisors数组
    updateAdvisorArray();
    //通知监听器
    adviceChanged();
}
```

然后看下关于引介增强器的添加addAdvisor，我们知道引介就是对目标类增加新的接口，所以引介增强，也就是对接口的处理：

```
public void addAdvisor(int pos, IntroductionAdvisor advisor) throws AopConfigException {
	//对接口进行校验
    advisor.validateInterfaces();

    // 遍历要添加的接口，添加
    for (int i = 0; i < advisor.getInterfaces().length; i++) {
    	//就是添加到interfaces集合中，interfaces是一个HashSet
        addInterface(advisor.getInterfaces()[i]);
    }
    //然后添加到advisors中
    addAdvisorInternal(pos, advisor);
}
```

对于添加增强的步骤，就是把我们的增强器添加进代理工厂中，保存在一个LinkedList中，顺序是添加进来的顺序。

## 获取代理
到目前为止，我们看到的都还是在组装代理工厂，并没有看到代理的生成，接下来`proxyFactory.getProxy()`这一步就是获取代理的过程，我们继续看ProxyFactory的getProxy方法：

```
public Object getProxy() {
	//创建一个AOP代理
    AopProxy proxy = createAopProxy();
    //返回代理
    return proxy.getProxy();
}
```
我们知道一般创建代理会有两种方式，一种是JDK动态代理，另外一种是CGLIB动态代理，而这里的创建AOP代理就是生成这两种代理中的一种。先看createAopProxy()方法，在AdvisedSupport类中：

```
protected synchronized AopProxy createAopProxy() {
    if (!this.isActive) {
        activate();
    }
    //获取AOP代理工厂，然后创建代理
    return getAopProxyFactory().createAopProxy(this);
}
```

获取代理工厂这一步，这里就是默认获取一个DefaultAopProxyFactory实例，然后调用createAopProxy创建AOP代理：

```
public AopProxy createAopProxy(AdvisedSupport advisedSupport) throws AopConfigException {
	//对于指定了使用CGLIB方式，或者代理的是类，或者代理的不是接口，就使用CGLIB的方式来创建代理
    boolean useCglib = advisedSupport.getOptimize() || advisedSupport.getProxyTargetClass() || advisedSupport.getProxiedInterfaces().length == 0;
    if (useCglib) {
        return CglibProxyFactory.createCglibProxy(advisedSupport);
    }
    else {
        //使用JDK动态代理来创建代理
        return new JdkDynamicAopProxy(advisedSupport);
    }
}
```
获取完AOP代理之后返回，然后就是调用getProxy方法获取代理，这里分为CGLIB的获取方式和JDK动态代理的获取方式两种。

### JDK动态代理方式获取代理
JDK动态代理方式获取代理，实现在JdkDynamicAopProxy中：

```
public Object getProxy() {
    return getProxy(Thread.currentThread().getContextClassLoader());
}
```

```
public Object getProxy(ClassLoader cl) {
	//JDK动态代理只能代理接口类型，先获取接口
    //就是从AdvisedSupport中获取保存在interfaces中的接口
    Class[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advisedSupport);
    //使用Java的反射机制创建一个代理实例
    return Proxy.newProxyInstance(cl, proxiedInterfaces, this);
}
```
关于JDK反射创建代理之类的，这里不做解析。

### CGLIB方式获取代理
CGLIB获取方式，实现在Cglib2AopProxy中：

```
public Object getProxy() {
	//使用CGLIB的方式来获取，CGLIB这里不做解析
    return getProxy(Thread.currentThread().getContextClassLoader());
}
```

## 使用代理
上面获取代理之后，就剩最后一步，使用，当我们调用业务方法的时候，实际上是调用代理中的方法，对于CGLIB生成的代理，调用的是DynamicAdvisedInterceptor的intercept方法；JDK动态代理生成的代理是调用invoke方法。

### JDK动态代理
看下JDK动态代理的方式，对于方法的调用，实际上调用的是代理类的invoke方法，在JdkDynamicAopProxy中：

```
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    MethodInvocation invocation = null;
    Object oldProxy = null;
    boolean setProxyContext = false;
	//代理的目标对象
    TargetSource targetSource = advisedSupport.targetSource;
    Class targetClass = null;
    Object target = null;		

    try {
        //equals方法
        if (method.getDeclaringClass() == Object.class && "equals".equals(method.getName())) {
            return equals(args[0]) ? Boolean.TRUE : Boolean.FALSE;
        }
        else if (Advised.class == method.getDeclaringClass()) {
            //？？？
            return AopProxyUtils.invokeJoinpointUsingReflection(this.advisedSupport, method, args);
        }

        Object retVal = null;

        //代理目标对象
        target = targetSource.getTarget();
        if (target != null) {
            targetClass = target.getClass();
        }
		//？？？
        if (this.advisedSupport.exposeProxy) {
            // Make invocation available if necessary
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        //获取配置的通知Advicelian
        List chain = this.advisedSupport.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                this.advisedSupport, proxy, method, targetClass);

        //没有配置通知
        if (chain.isEmpty()) {
            //直接调用目标对象的方法
            retVal = AopProxyUtils.invokeJoinpointUsingReflection(target, method, args);
        }
        else {
            //配置了通知，创建一个MethodInvocation
            invocation = new ReflectiveMethodInvocation(proxy, target,
                                method, args, targetClass, chain);

            //执行通知链，沿着通知器链调用所有的通知
            retVal = invocation.proceed();
        }

        //返回值
        if (retVal != null && retVal == target) {
            //返回值为自己
            retVal = proxy;
        }
        //返回
        return retVal;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            targetSource.releaseTarget(target);
        }

        if (setProxyContext) {
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```

### CGLIB动态代理
CGLIB的是调用DynamicAdvisedInterceptor的intercept方法对目标对象进行处理，具体暂先不解析。


# 使用ProxyFactoryBean创建AOP代理
ProxyFactoryBean对Pointcut和Advice提供了完全的控制，还包括应用的顺序。ProxyFactoryBean的getObject方法会返回一个AOP代理，包装了目标对象。

Spring在初始化的过程中，createBean的时候，如果是FactoryBean的话，会调用` ((BeanFactoryAware)bean).setBeanFactory(this);`：

```
public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
    this.beanFactory = beanFactory;
    //创建通知器链
    this.createAdvisorChain();
    if(this.singleton) {
    	//刷新目标对象
        this.targetSource = this.freshTargetSource();
        //获取单例实例
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

createAopProxy方法在AdvisedSupport类中，下面创建的流程跟上面解析的都一样了。

到这里AOP的一个流程的源码算是走完了，这只是其中一小部分，还有很多的没有涉及到，包括AOP标签的解析，CGLIB生成代理以及调用代理等等。其中有些还没明白的已经画上了问号，慢慢的在研究下。

