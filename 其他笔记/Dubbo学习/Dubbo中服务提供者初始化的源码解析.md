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

Spring容器启动后，遇到自定义标签，会使用相应的标签解析器进行解析标签，比如遇到了dubbo的service标签后，就会使用`com.alibaba.dubbo.config.spring.schema.DubboBeanDefinitionParser`来解析service标签，对应的bean是`com.alibaba.dubbo.config.spring.ServiceBean`，标签解析完成后Spring容器就会存在一个ServiceBean，我们先看下ServiceBean的相关代码：

```java
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener, BeanNameAware {
    ...
}
```

ServiceBean实现了Spring的若干接口，会在容器启动以及Bean实例化的时候依次执行相关方法：

1. BeanNameAware接口，会调用setBeanName方法设置bean的名字。
2. ApplicationContextAware接口，会调用setApplicationContext方法设置上下文。
3. InitializingBean接口，

