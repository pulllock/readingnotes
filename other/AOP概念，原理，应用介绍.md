心情没法不沉重，被问到AOP是什么？AOP原理是什么？我竟然张大了嘴巴，说不出来！对于一个程序员的打击，还能有比这更大的吗？我没脸说我是个写代码的，我也没脸说我是程序员。

# AOP是什么？
## 定义
AOP，面向切面编程，是对OOP的补充。从网上看到的一句话：`这种在运行时，动态的将代码切入到类的指定方法或者指定位置上的编程思想，就是面向切面的编程。`这是其中的一种方式，在运行时动态添加。还有另外一种是在编译代码的时候，将代码切入到指定的方法或者位置上去，这是静态添加的方式。

## 使用

我们在实际的业务中都会有一些公共逻辑，比如日志的记录，事务的管理等等，而如果每次都把日志和事务的代码手动写到业务逻辑前后，重复代码就相当可怕，而如果这些额外代码有修改，必须要每个都修改，这是相当不明智的。AOP可以帮我们解决这些问题。

## 实现

其实AOP本身并不能帮我们解决那些问题，AOP就是一种思想，而帮我们解决的是具体的AOP的实现，比如aspectj，jboss AOP，以及我们最熟悉的Spring AOP，Spring AOP在Spring1.0的时候是自己实现的AOP框架，在2.0之后就开始集成了aspectj。现在我们所说的Spring AOP就是Spring加Aspectj这种方式。

# AOP的相关概念
对于AOP中相关的概念，我们接触更多的还是Spring AOP，这里主要是以Spring AOP的概念来说明：

- Aspect，切面，一个关注点的模块化，这个关注点可能会横切多个对象。
- JoinPoint，连接点，在程序执行过程中某个特定的点，比如某方法调用的时候或者处理异常的时候。在Spring AOP中，一个连接点总是表示一个方法的执行。
- Advice，通知，在切面的某个特定的连接点上执行的动作。
- Pointcut，切点，匹配连接点的断言。通知和一个切入点表达式关联，并在满足这个切入点的连接点上运行（例如，当执行某个特定名称的方法时）。切入点表达式如何和连接点匹配是AOP的核心：Spring缺省使用AspectJ切入点语法。

上面是关于AOP中几个基本概念的定义，下面看下有关我们使用时的一些概念：

- Target Object，目标对象，被一个或者多个切面所通知的对象。也就是我们业务中实际要进行增强的业务对象。
- AOP Proxy，AOP代理，AOP框架创建的对象。也就是被增强之后的对象。
- Weaving，织入，把切面连接到其它的应用程序类型或者对象上，并创建一个被通知的对象。就是把切面作用到目标对象，然后产生一个代理对象的过程。

还有另外一个概念，是给类声明额外方法的概念：

- Introduction，引介，用来给一个类型声明额外的方法或属性。就是我可以不用实现另外一个接口，就能使用那个接口的方法。

## 通知类型
Advice是通知，也就是在切面的某个连接点上要执行的动作，也就是我们要编写的增强功能的代码。通知也分为好几种类型，分别有不同作用：

- 前置通知（Before advice）：在某连接点之前执行的通知，但这个通知不能阻止连接点之前的执行流程（除非它抛出一个异常）。
- 后置通知（After returning advice）：在某连接点正常完成后执行的通知：例如，一个方法没有抛出任何异常，正常返回。
- 异常通知（After throwing advice）：在方法抛出异常退出时执行的通知。
- 最终通知（After (finally) advice）：当某连接点退出的时候执行的通知（不论是正常返回还是异常退出）。
- 环绕通知（Around Advice）：包围一个连接点的通知，如方法调用。这是最强大的一种通知类型。环绕通知可以在方法调用前后完成自定义的行为。它也会选择是否继续执行连接点或直接返回它自己的返回值或抛出异常来结束执行。

# AOP原理
上面说到了AOP可以在编译时候将代码织入到指定的方法或者属性上，或者在运行的时候动态的将代码切入到指定的方法或者属性中，这描述了AOP应该要做的事情，其实也基本算是它的原理了，AOP实现的关键就是创建AOP代理，代理有静态代理和动态代理之分，其中aspectj为静态代理，Spring AOP是动态代理，这里把静态和运行时动态的分开说。

## AspectJ编译时增强
aspectj编译时增强，既是静态代理增强，也就是会在编译阶段生成代理，将代码织入到Java的字节码中去。

## Spring AOP的运行时增强
Spring AOP是基于代理机制的，并且Spring AOP使用的是动态代理增强，动态代理不会改变类的字节码，而是动态的生成代理对象。Spring AOP的动态代理机制有两种方式：JDK动态代理和CGLIB动态代理。

### JDK动态代理
JDK动态代理需要被代理的类必须实现一个接口，通过使用反射来接受被代理的类。
### CGLIB动态代理
CGLIB动态代理可以不用需要被代理类必须实现接口，被代理类可以是一个类。

# Spring AOP
上面说到了Spring AOP中使用的是动态代理机制，同时也分为两种代理机制：JDK动态代理和CGLIB动态代理，这可以在源码中看到，在DefaultAopProxyFactory的createAopProxy中可看到：

```
public AopProxy createAopProxy(AdvisedSupport advisedSupport) throws AopConfigException {
	//可以看到这里有条件是没有实现接口
    boolean useCglib = advisedSupport.getOptimize() || advisedSupport.getProxyTargetClass() || advisedSupport.getProxiedInterfaces().length == 0;
    if (useCglib) {
        return CglibProxyFactory.createCglibProxy(advisedSupport);
    }
    else {
        // Depends on whether we have expose proxy or frozen or static ts
        return new JdkDynamicAopProxy(advisedSupport);
    }
}
```

对于接口的代理使用的JDK动态代理，而对于类的代理使用的是CGLIB动态代理。

# AOP的使用场景
AOP适用于具有横切逻辑的应用，比如性能监控，日志记录，缓存，事务管理，访问控制等。

有关Spring AOP的例子可以参考[Spring中AOP的配置从1.0到5.0的演进](http://cxis.me/2017/04/10/Spring%E4%B8%ADAOP%E7%9A%84%E9%85%8D%E7%BD%AE%E4%BB%8E1.0%E5%88%B05.0%E7%9A%84%E6%BC%94%E8%BF%9B/)，这里有具体的配置。