最近在学习Spring稍微深入一点的东西，在这过程中发现虽然有很多关于各种AOP，IOC原理配置等的文章，但是都只是针对某一版本或者压根儿就没有标明版本的解析配置等。或许是我理解力不够，为了方便自己以后快速找到这些东西去看，还是自己记录下。

这里主要是记录下从Spring1.0到现在的5.0中AOP的配置方式，关于AOP原理和源码，暂先不解释。主要用作自己记录用，如果有错误的还请指出一起改正学习，免得误导别人，谢谢。

# Spring1中AOP的配置
直接看Spring1.1.1的文档，里面都已经给出来了各种配置方式，更高版本的也都包含了这些，但是觉得看1.1.1的更纯粹一些。

## 使用ProxyFactoryBean创建AOP代理

使用ProxyFactoryBean的方式来配置AOP，是最基础的方法。这里我们用的是代理接口的方式，步骤大概是：

1. 定义我们的业务接口和业务实现类。
2. 定义通知类，就是我们要对目标对象进行增强的类。
3. 定义ProxyFactoryBean，这里封装了AOP的功能。

这就是Spring中AOP的配置，这种方式简单明了，往下接着看示例代码。

我们的业务接口，LoginService：

```
package me.cxis.spring.aop;

/**
 * Created by cheng.xi on 2017-03-29 12:02.
 */
public interface LoginService {
    String login(String userName);
}
```

业务接口的实现类，LoginServiceImpl：

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
在登录前做一些操作的通知类，LogBeforeLogin：

```
//这里只是在登录方法调用之前打印一句话
public class LogBeforeLogin implements MethodBeforeAdvice {
    public void before(Method method, Object[] objects, Object o) throws Throwable {
        System.out.println("有人要登录了。。。");
    }
}
```

配置文件，在Spring1.x中主要还是用xml的方式来配置：

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
        <!--要代理的接口-->
        <property name="proxyInterfaces">
            <value>me.cxis.spring.aop.LoginService</value>
        </property>
        <!--拦截器名字，也就是我们定义的通知类-->
        <property name="interceptorNames">
            <list>
                <value>logBeforeLogin</value>
            </list>
        </property>
        <!--目标类，就是我们业务的实现类-->
        <property name="target">
            <ref bean="loginServiceImpl"/>
        </property>
    </bean>
</beans>
```

测试方法，Main：

```
package me.cxis.spring.aop;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * Created by cheng.xi on 2017-03-29 10:34.
 */
public class Main {


    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:aop.xml");
        LoginService loginService = (LoginService) applicationContext.getBean("loginProxy");
        loginService.login("sdf");
    }
}
```

上面的例子是用的是代理接口的方式，关于代理类的方式这里不做介绍，代理类的方式需要依赖CGLIB。从上面的代码可以看到，这种方式使用AOP简单直观，也是我们理解Spring AOP原理的很好的入口，但是在使用的时候，可能会发现业务增多了之后，ProxyFactoryBean的配置也会增多，导致xml迅速变多。

另外这种方式的使用会把LoginService接口中所有的方法都代理了，也就是说每个方法都会被增强，如果不想被增强，还可以使用另外一种方式，配置Advisor。

### 使用ProxyFactoryBean和Advisor的方式创建AOP代理
使用Advisor配合，可以指定要增强的方法，不会把整个类中的所有方法都代理了。

这种方式跟上面的方式相比较，只是xml配置文件发生了变化，其他的代码都没有变，所以这里只列出了xml代码，其他的参照上面的代码。

xml配置：

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans>
    <!--业务处理类，也就是被代理的类-->
    <bean id="loginServiceImpl" class="me.cxis.spring.aop.advisor.LoginServiceImpl"/>

    <!--通知类-->
    <bean id="logBeforeLogin" class="me.cxis.spring.aop.advisor.LogBeforeLogin"/>

    <!--切面-->
    <bean id="loginAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
        <property name="advice">
            <ref bean="logBeforeLogin"/>
        </property>
        <property name="pattern">
            <value>me.cxis.spring.aop.advisor.LoginServiceImpl.login*</value>
        </property>
    </bean>

    <!--代理类-->
    <bean id="loginProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="interceptorNames">
            <list>
                <value>loginAdvisor</value>
            </list>
        </property>
        <property name="target">
            <ref bean="loginServiceImpl"/>
        </property>
    </bean>
</beans>
```
可以看到这里我们多了Advisor，Advisor中可以使用正则表达式来匹配要增强的方法。

## 使用ProxyFactory编程的方式创建AOP代理
也可以使用直接代码的方式，不依赖xml文件，来创建AOP代理，直接看示例代码。

业务接口，LoginService：

```
package me.cxis.spring.aop.proxyfactory;

/**
 * Created by cheng.xi on 2017-03-29 12:02.
 */
public interface LoginService {
    String login(String userName);
}
```

业务接口实现类，LoginServiceImpl：

```
package me.cxis.spring.aop.proxyfactory;

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
通知类，LogBeforeLogin：

```
package me.cxis.spring.aop.proxyfactory;

import org.springframework.aop.MethodBeforeAdvice;

import java.lang.reflect.Method;

/**
 * Created by cheng.xi on 2017-03-29 10:56.
 */
public class LogBeforeLogin implements MethodBeforeAdvice {
    public void before(Method method, Object[] objects, Object o) throws Throwable {
        System.out.println("有人要登录了。。。");
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

        LoginService loginService = (LoginService) proxyFactory.getProxy();//从代理工厂中获取代理
        loginService.login("x");
    }
}
```

这种方式跟上面使用ProxyFactoryBean的方式差不多，步骤也基本相同，不做过多解释。Spring不推荐这种方式。

## 使用autoproxy方式创建AOP
使用这种方式创建AOP代理，最主要的是可以使用一个配置来代理多个业务bean，也就是跟上面使用ProxyFactoryBean不同的地方，ProxyFactoryBean需要配置很多个ProxyFactoryBean配置，而autoproxy相对会很少。

使用autoproxy也有两种方式：BeanNameAutoProxyCreator和DefaultAdvisorAutoProxyCreator。两种方式的差别直接看代码。

### BeanNameAutoProxyCreator方式
业务接口，LoginService：

```
package me.cxis.spring.aop.autoproxy;

/**
 * Created by cheng.xi on 2017-03-29 12:02.
 */
public interface LoginService {
    String login(String userName);
}
```

业务接口实现类，LoginServiceImpl：

```
package me.cxis.spring.aop.autoproxy;

/**
 * Created by cheng.xi on 2017-03-29 10:36.
 */
public class LoginServiceImpl implements LoginService {

    public String login(String userName){
        System.out.println("autoproxy:正在登录");
        return "success";
    }
}
```

通知类，LogBeforeLogin：

```
package me.cxis.spring.aop.autoproxy;

import org.springframework.aop.MethodBeforeAdvice;

import java.lang.reflect.Method;

/**
 * Created by cheng.xi on 2017-03-29 10:56.
 */
public class LogBeforeLogin implements MethodBeforeAdvice {
    public void before(Method method, Object[] objects, Object o) throws Throwable {
        System.out.println("autoproxy:有人要登录了。。。");
    }
}
```

xml配置文件：

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans>
    <!--业务处理类，也就是被代理的类-->
    <bean id="loginService" class="me.cxis.spring.aop.autoproxy.LoginServiceImpl"/>

    <!--通知类-->
    <bean id="logBeforeLogin" class="me.cxis.spring.aop.autoproxy.LogBeforeLogin"/>

    <!--代理类-->
    <bean id="loginServiceProxy" class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">

        <property name="interceptorNames">
            <list>
                <value>logBeforeLogin</value>
            </list>
        </property>
        <property name="beanNames">
            <value>loginService*</value>
        </property>
    </bean>
</beans>
```

测试方法，Main：

```
package me.cxis.spring.aop.autoproxy;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * Created by cheng.xi on 2017-03-29 10:34.
 */
public class Main {

    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:aop-auto-proxy.xml");
        LoginService loginService = (LoginService) applicationContext.getBean("loginService");
        loginService.login("sdf");
    }
}
```
在xml配置中beanNames的值我们使用了通配符，也就是我们可以使用这一个BeanNameAutoProxyCreator来匹配很多个接口，在ProxyFactoryBean的方式中，我们则需要配置很多ProxyFactoryBean的配置。

另外也需要注意下在测试方法中我们对bean的调用方式，之前我们是调用代理类，现在我们直接调用的Bean。这种方式中也还是把业务类中的所有的方法都增强了。

### DefaultAdvisorAutoProxyCreator方式
业务接口，LoginService：

```
package me.cxis.spring.aop.advisorautoproxy;

/**
 * Created by cheng.xi on 2017-03-29 12:02.
 */
public interface LoginService {
    String login(String userName);
}
```

业务接口实现类，LoginServiceImpl：

```
package me.cxis.spring.aop.advisorautoproxy;

/**
 * Created by cheng.xi on 2017-03-29 10:36.
 */
public class LoginServiceImpl implements LoginService {

    public String login(String userName){
        System.out.println("advisorautoproxy:正在登录");
        return "success";
    }
}

```

通知类，LogBeforeLogin：

```
package me.cxis.spring.aop.advisorautoproxy;

import org.springframework.aop.MethodBeforeAdvice;

import java.lang.reflect.Method;

/**
 * Created by cheng.xi on 2017-03-29 10:56.
 */
public class LogBeforeLogin implements MethodBeforeAdvice {
    public void before(Method method, Object[] objects, Object o) throws Throwable {
        System.out.println("advisorautoproxy:有人要登录了。。。");
    }
}
```

配置文件xml：

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans>
    <!--业务处理类，也就是被代理的类-->
    <bean id="loginService" class="me.cxis.spring.aop.advisorautoproxy.LoginServiceImpl"/>

    <!--通知类-->
    <bean id="logBeforeLogin" class="me.cxis.spring.aop.advisorautoproxy.LogBeforeLogin"/>

    <bean id="logBeforeAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
        <property name="advice">
            <ref bean="logBeforeLogin"/>
        </property>
        <property name="pattern">
            <value>me.cxis.spring.aop.advisorautoproxy.*</value>
        </property>
    </bean>

    <!--代理类-->
    <bean id="advisorAutoProxy" class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator">
    </bean>
</beans>
```

测试方法：

```
package me.cxis.spring.aop.advisorautoproxy;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * Created by cheng.xi on 2017-03-29 10:34.
 */
public class Main {


    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:aop-advisor-auto-proxy.xml");
        LoginService loginService = (LoginService) applicationContext.getBean("loginService");
        loginService.login("sdf");
    }
}

```

在这里的配置文件中我们使用了正则的方式来配置Advisor，这种可以匹配指定的方法，不需要把类中的所有的方法都增强了。

## 引介增强Introduction
上面所有AOP的代理都是对方法的增强，而引介增强则是对类的增强，所谓对类的增强就是，我是A类，实现了接口B，但是我没有实现接口C，那么通过引介增强的方式，我没有实现接口C，但是我可以调用C中的方法。

业务接口，LoginService：

```
package me.cxis.spring.aop.introduction;

/**
 * Created by cheng.xi on 2017-03-29 12:02.
 */
public interface LoginService {
    String login(String userName);
}
```

业务接口实现类：

```
package me.cxis.spring.aop.introduction;

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

另外一个业务接口，SendEmailService：

```
package me.cxis.spring.aop.introduction;

/**
 * Created by cheng.xi on 2017-03-30 23:45.
 */
public interface SendEmailService {
    void sendEmail();
}

```

增强，LogAndSendEmailBeforeLogin：

```
package me.cxis.spring.aop.introduction;

import org.aopalliance.intercept.MethodInvocation;
import org.springframework.aop.MethodBeforeAdvice;
import org.springframework.aop.support.DelegatingIntroductionInterceptor;

import java.lang.reflect.Method;

/**
 * Created by cheng.xi on 2017-03-29 10:56.
 */
public class LogAndSendEmailBeforeLogin extends DelegatingIntroductionInterceptor implements SendEmailService {


    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        System.out.println("有人要登录了。。。");
        return super.invoke(mi);
    }

    public void sendEmail() {
        System.out.println("发送邮件。。。。");
    }
}
```

xml配置文件：

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans>
    <!--业务处理类，也就是被代理的类-->
    <bean id="loginServiceImpl" class="me.cxis.spring.aop.introduction.LoginServiceImpl"/>

    <!--通知类-->
    <bean id="logAndSendEmailBeforeLogin" class="me.cxis.spring.aop.introduction.LogAndSendEmailBeforeLogin"/>

    <!--代理类-->
    <bean id="loginProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="interfaces">
            <value>me.cxis.spring.aop.introduction.SendEmailService</value>
        </property>
        <property name="interceptorNames">
            <list>
                <value>logAndSendEmailBeforeLogin</value>
            </list>
        </property>
        <property name="target">
            <ref bean="loginServiceImpl"/>
        </property>
        <property name="proxyTargetClass">
            <value>true</value>
        </property>
    </bean>
</beans>
```

测试方法：

```
package me.cxis.spring.aop.introduction;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * Created by cheng.xi on 2017-03-29 10:34.
 */
public class Main {

    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:aop-introduction.xml");
        LoginServiceImpl loginService = (LoginServiceImpl) applicationContext.getBean("loginProxy");
        loginService.login("sdf");

        SendEmailService sendEmailService = (SendEmailService) loginService;
        sendEmailService.sendEmail();
    }
}

```
这样就可以了～

# Spring2中AOP的配置
What's new in Spring 2.0?

- 添加了基于schema的AOP支持。
- 加入了AspectJ的支持，添加了@AspectJ注解。

上面就是AOP在2.0版本新增的特性，1.0的所有AOP配置方式在2.0中都支持，下面主要看看2.0中新增的一些方法。

## 使用@AspectJ的方式配置AOP代理
步骤大概如下：

1. 定义我们的业务接口和业务实现类。
2. 定义通知类，就是我们要对目标对象进行增强的类。使用@AspectJ注解。
3. 启用@AspectJ支持。

还是看代码。

业务接口，LoginService：

```
package me.cxis.spring.aop;

/**
 * Created by cheng.xi on 2017-03-29 12:02.
 */
public interface LoginService {
    String login(String userName);
}
```

业务接口实现，LoginServiceImpl：

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

增强类，LogBeforeLogin，请注意这里面使用了注解：

```
package me.cxis.spring.aop;

import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;

/**
 * Created by cheng.xi on 2017-03-29 10:56.
 */
@Aspect
public class LogBeforeLogin {

    @Pointcut("execution(* me.cxis.spring.aop.*.login(..))")
    public void loginMethod(){}

    @Before("loginMethod()")
    public void beforeLogin(){
        System.out.println("有人要登录了。。。");
    }
}
```

xml配置文件：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <!--@AspectJ支持-->
    <aop:aspectj-autoproxy/>

    <!--业务实现-->
    <bean id="loginService" class="me.cxis.spring.aop.LoginServiceImpl"/>

    <!--Aspect-->
    <bean id="logBeforeLoginAspect" class="me.cxis.spring.aop.LogBeforeLogin">
    </bean>

</beans>
```

测试方法：

```
package me.cxis.spring.aop;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * Created by cheng.xi on 2017-03-29 10:34.
 */
public class Main {

    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:aop.xml");
        LoginService loginService = (LoginService) applicationContext.getBean("loginService");
        loginService.login("sdf");
    }
}
```

可以看下上面的配置文件，首先开启@AspectJ注解的支持，然后只需要声明一下业务bean和增强bean，其余的都不用做了，是不是比以前方便多了。以前的那些配置，全部都在增强类中用注解处理了。

### 使用自定义注解作为execution的表达式
上面使用普通的execution表达式来声明对那些方法进行增强，也可以使用注解的方式，其实就是把表达式换成了注解，只有添加了注解的方法才能被增强。

先定义一个注解UseAop：

```
package me.cxis.spring.aop.customannotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Created by cheng.xi on 2017-03-31 11:29.
 * 标注此注解的方法，需要使用AOP代理
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface UseAop {
}
```

业务接口，LoginService：

```
package me.cxis.spring.aop.customannotation;

/**
 * Created by cheng.xi on 2017-03-29 12:02.
 */
public interface LoginService {
    String login(String userName);
}
```

业务接口的实现，LoginServiceImpl：

```
package me.cxis.spring.aop.customannotation;

/**
 * Created by cheng.xi on 2017-03-29 10:36.
 */
public class LoginServiceImpl implements LoginService {

    @UseAop
    public String login(String userName){
        System.out.println("正在登录");
        return "success";
    }
}
```
这里业务实现类的方法使用了注解，表明这个方法需要使用AOP代理。

增强类，LogBeforeLogin：

```
@Aspect
public class LogBeforeLogin {
	
    @Pointcut("@annotation(me.cxis.spring.aop.customannotation.UseAop)")
    public void loginMethod(){}

    @Before("loginMethod()")
    public void beforeLogin(){
        System.out.println("有人要登录了。。。");
    }
}
```
这里是把表达式换成了我们定义的注解。

xml配置：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <!--@AspectJ支持-->
    <aop:aspectj-autoproxy/>
    
    <!--业务实现-->
    <bean id="loginService" class="me.cxis.spring.aop.customannotation.LoginServiceImpl"/>

    <!--Aspect-->
    <bean id="logBeforeLoginAspect" class="me.cxis.spring.aop.customannotation.LogBeforeLogin">
    </bean>

</beans>
```
xml配置文件并没有改变。同样下面的测试方法也没有变。

测试方法：

```
package me.cxis.spring.aop.customannotation;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * Created by cheng.xi on 2017-03-29 10:34.
 */
public class Main {

    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:aop-annotation.xml");
        LoginService loginService = (LoginService) applicationContext.getBean("loginService");
        loginService.login("sdf");
    }
}
```

## 使用基于schema的方式配置AOP代理
基于schema的方式配置，可以不使用注解，而是完全基于xml配置。

业务接口，实现类，增强如下：

```
package me.cxis.spring.aop.config;

/**
 * Created by cheng.xi on 2017-03-29 12:02.
 */
public interface LoginService {
    String login(String userName);
}

package me.cxis.spring.aop.config;

/**
 * Created by cheng.xi on 2017-03-29 10:36.
 */
public class LoginServiceImpl implements LoginService {

    public String login(String userName){
        System.out.println("正在登录");
        return "success";
    }
}

package me.cxis.spring.aop.config;

/**
 * Created by cheng.xi on 2017-03-29 10:56.
 */
public class LogBeforeLogin {

    public void beforeLogin(){
        System.out.println("有人要登录了。。。");
    }
}
```
可以看到上面三个类中，增强类只是一个普通的bean而已，所有的配置都在xml中。

xml配置：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <!--业务实现类-->
    <bean id="loginService" class="me.cxis.spring.aop.config.LoginServiceImpl"></bean>

    <!--增强类-->
    <bean id="logBeforeLogin" class="me.cxis.spring.aop.config.LogBeforeLogin"></bean>

    <aop:config>
        <aop:aspect id="loginAspect" ref="logBeforeLogin">
            <aop:pointcut expression="execution(* me.cxis.spring.aop.config.*.*(..))" id="beforeLoginPointCut"/>
            <aop:before method="beforeLogin" pointcut-ref="beforeLoginPointCut"/>
        </aop:aspect>
    </aop:config>

</beans>
```
可以类比下上面@AspectJ的方式，其实是一样的，只不过把注解的东西都放到了xml中。

测试方法：

```
package me.cxis.spring.aop.config;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * Created by cheng.xi on 2017-03-29 10:34.
 */
public class Main {

    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:aop-config.xml");
        LoginService loginService = (LoginService) applicationContext.getBean("loginService");
        loginService.login("sdf");
    }
}
```

## 使用dtd方式
我们上面看到使用`<aop:aspectj-autoproxy/>`来启用@AspectJ的支持，这种方式使用的基于schema扩展的，如果想用原来的DTD模式也是可以的，使用AnnotationAwareAspectJAutoProxyCreator即可。

业务接口，业务实现类，增强类的代码都跟`使用@AspectJ的方式配置AOP代理`这一步的一样，除了xml里面有点不一样，代码如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd ">

    <bean class="org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator"/>

    <!--业务实现-->
    <bean id="loginService" class="me.cxis.spring.aop.dtd.LoginServiceImpl"/>

    <!--Aspect-->
    <bean id="logBeforeLoginAspect" class="me.cxis.spring.aop.dtd.LogBeforeLogin">
    </bean>

</beans>
```
就是将`<aop:aspectj-autoproxy/>`替换成`<bean class="org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator"/>`，Spring文档上也有说明。

## 编程的方式使用@AspectJ
业务接口LoginService：

```
package me.cxis.spring.aop.programmatic;

/**
 * Created by cheng.xi on 2017-03-29 12:02.
 */
public interface LoginService {
    String login(String userName);
}
```

业务实现类：

```
package me.cxis.spring.aop.programmatic;

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

增强类：

```
package me.cxis.spring.aop.programmatic;

import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;


/**
 * Created by cheng.xi on 2017-03-29 10:56.
 */
@Aspect
public class LogBeforeLogin {

    @Pointcut("execution(* me.cxis.spring.aop.programmatic.*.login(..))")
    public void loginMethod(){}

    @Before("loginMethod()")
    public void beforeLogin(){
        System.out.println("有人要登录了。。。");
    }
}

```

测试方法：

```
package me.cxis.spring.aop.programmatic;

import org.springframework.aop.aspectj.annotation.AspectJProxyFactory;

/**
 * Created by cheng.xi on 2017-03-29 10:34.
 */
public class Main {


    public static void main(String[] args) {

        AspectJProxyFactory factory = new AspectJProxyFactory();//创建代理工厂
        factory.setTarget(new LoginServiceImpl());//设置目标类
        factory.addAspect(LogBeforeLogin.class);//设置增强
        LoginService loginService = factory.getProxy();//获取代理
        loginService.login("xsd");
    }
}
```
基本跟使用xml方式的@Aspect的步骤差不多。

## 引介增强Introduction
感觉引介增强比1.0更强大了点，直接看代码，对比1.0的引介就知道了。

业务接口：

```
package me.cxis.spring.aop.introduction;

/**
 * Created by cheng.xi on 2017-03-29 12:02.
 */
public interface LoginService {
    String login(String userName);
}
```

接口实现：

```
package me.cxis.spring.aop.introduction;

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

另外一个业务接口：

```
package me.cxis.spring.aop.introduction;

/**
 * Created by cheng.xi on 2017-03-30 23:45.
 */
public interface SendEmailService {
    void sendEmail();
}
```
接口实现：

```
package me.cxis.spring.aop.introduction;

/**
 * Created by cheng.xi on 2017-03-31 13:46.
 */
public class SendMailServiceImpl implements SendEmailService {
    public void sendEmail() {
        System.out.println("发送邮件。。。。");
    }
}
```

增强，也就是把上面两个接口关联起来的地方：

```
package me.cxis.spring.aop.introduction;

import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.DeclareParents;

/**
 * Created by cheng.xi on 2017-03-29 10:56.
 */
@Aspect
public class LogAndSendEmailBeforeLogin {

    @DeclareParents(value = "me.cxis.spring.aop.introduction.LoginServiceImpl",defaultImpl = SendMailServiceImpl.class)
    private SendEmailService sendEmailService;
}
```

xml配置：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <!--@AspectJ支持-->
    <aop:aspectj-autoproxy/>

    <!--业务实现-->
    <bean id="loginService" class="me.cxis.spring.aop.introduction.LoginServiceImpl"/>

    <!--Aspect-->
    <bean id="logBeforeLoginAspect" class="me.cxis.spring.aop.introduction.LogAndSendEmailBeforeLogin">
    </bean>

</beans>
```

测试方法：

```
package me.cxis.spring.aop.introduction;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * Created by cheng.xi on 2017-03-29 10:34.
 */
public class Main {

    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:aop-introduction.xml");
        LoginService loginService = (LoginService) applicationContext.getBean("loginService");
        loginService.login("sdf");

        SendEmailService sendEmailService = (SendEmailService) loginService;
        sendEmailService.sendEmail();
    }
}
```
具体的Introduction不做过多解释。另外上面也都是基于接口进行的代理，关于基于类的，这里不做说明。

上面是Spring2.0新增的关于AOP的配置的东西，1.0的方式在2.0中仍然适用。另外在Spring2.5中还增加了AspectJ的load-time织入的支持，也就是在类加载的时候织入。

# Spring3,4,5中AOP的配置
Spring3,4,5基本就没在增加新的配置方式了，使用的方式基本都还是1.0和2.0中的方式，但是还会有很多的细节以及小特性添加，这里不能过多深入理解。暂先到这里。