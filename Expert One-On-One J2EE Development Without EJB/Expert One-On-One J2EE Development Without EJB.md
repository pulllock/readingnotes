# 轻量级容器与控制反转

## 轻量级容器
任何容器都应该提供以下服务：

- 生命周期管理，容器用于控制应用对象运行的生命周期，最起码容器必须将创建新对象的逻辑从使用者那里抽象出来。
- 查找服务，容器应该提供某种途径，用于获得受管对象的引用。查找服务是容器的核心服务，也就是说，容器的核心就是一个工厂。
- 配置管理，容器需要提供统一的方法来配置运行其中的对象，并允许对象的参数化。
- 依赖决议，容器不仅可以管理String，int等简单类型的配置，还可以管理其中各个对象之间的关系。

## 控制反转
### IOC实现策略
IOC主要的实现形式有两种：

- 依赖查找，容器提供回调接口和上下文环境给组件，组件必须自己使用容器提供的API来查找资源和协作对象，容器调用回调方法，从而让应用代码获得相关资源。
- 依赖注入，组件不做定位查询，只提供普通的Java方法让容器去决定依赖关系。容器全权负责组件的装配，它会把符合依赖关系的对象通过JavaBean属性或者构造传递给需要的对象。setter方法注入和Constructor注入。

## IOC容器
### Spring框架

- 依赖检查，Spring可以检查是否已经装配了所有JavaBean属性，对象的依赖关系和简单属性可以分开检查。
- 自动装配，Spring可以在当前容器中自动挑选满足依赖关系的对象，并通过JavaBean属性将其提供给需要的对象。
- 支持集合，Spring可以自动装配List，Map，以及其他常用的Java集合对象类型。也支持集合类型任意嵌套。
- init和destory方法，这两个方法必须是无参数的方法。

Spring也定义了回调方法，用于实现基于回调，依赖查询式的IOC。不过要以引入依赖SpringAPI为代价。

### PicoContainer
略

# Spring框架简介
## 一个分层的应用框架
### 基础构建模块
Spring框架的关键概念：

- bean工厂，Spring轻量级IOC容器能配置，装配JavaBean和大多数普通Java对象，开发者不必定制Singleton和自己的配置机制。Spring提供了多个bean工厂实现。
- 应用上下文，application context是对bean工厂的扩展，在Bean工厂的基础上增加了对信息源和资源加载的支持，并提供了接入现有系统环境的能力。提供了多个上下文的实现。
- AOP框架，提供了对AOP的支持，可以对轻量级容器福安里的任何对象进行方法拦截。
- 自动代理，在AOP框架之上提供了更高级别的抽象，同时也提供了很多基础性的服务。
- 事务管理，Spring提供了通用的事务管理基础设施，包括可插接的事务策略和不同的事务边界划分方式。
- DAO的抽象，Spring定义了一组通用的数据访问异常类型，在创建通用的DAO接口时，可以用这些异常类型抛出有意义的异常信息，而不依赖于底层持久机制。
- JDBC的支持，Spring提供了两级JDBC抽象。集成了事务抽象和DAO的抽象。
- 集成O/R映射工具。提供了对多种O/R映射工具的支持。
- Web MVC框架，提供了DispatcherServlet和即插即用的控制器类。
- 远程调用支持。

## 核心Bean工厂
bean工厂用一种统一的方式去装配所有应用对象。借助反射和依赖注入，被bean工厂管理的组件根本不需要知道Spring的存在。
### 基础接口
Spring中最基础的接口是BeanFactory，提供了两个getBean方法，都可以根据String类型名查找获取bean实例，不同之处在于，其中一个getBean方法允许使用者检查获得的bean是否具有所需要的类型。

```
public interface BeanFactory {
    Object getBean(String name) throws BeansException;
	Object getBean(String name, Class requiredType) throws BeansException; 		
    boolean containsBean(String name);
	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
	String[] getAliases(String name) throws NoSuchBeanDefinitionException;
}
```
isSingleton()方法检测某个特定名称声明的bean被定义为Singleton还是Prototype。Singleton，所有getBean方法将返回同一个对象实例的引用；Prototype，每次getBean调用都会新建一个独立的对象实例。

getAliases()方法将返回它所有的别名。

BeanFactory的大多数实现知道自己处于整个工厂体系的什么位置，如果当前工厂中没有找到某个bean，就会向父工厂中查找，直到根工厂为止。

ListableBeanFactory子接口可以列出工厂中所有的bean，这个子接口提供了一系列方法，可以获取工厂中定义bean的数量，所有bean的名称，具有特定类型的所有bean的名称等。

```
public interface ListableBeanFactory extends BeanFactory {
	int getBeanDefinitionCount();
	String[] getBeanDefinitionNames();
	String[] getBeanDefinitionNames(Class type);
	boolean containsBeanDefinition(String name);
	Map getBeansOfType(Class type, boolean includePrototypes, boolean includeFactoryBeans) throws BeansException;
}
```

ListableBeanFactory可以获得被管理对象的相关信息。与BeanFactory不同的是，这个接口中的方法只可应用与当前的工厂实例，而不能进入工厂的层级体系。BeanFactoryUtils类也提供了类似的方法，可以遍历整个工厂体系。

Bean工厂和bean声明的读取是分离的，bean声明的解析是在读取器中实现的，可以与任何bean工厂配合使用。xml对应的读取器就是XmlBeanDefinitionReader。

如果在多线程环境下使用bean工厂，而其中一个bean的实现不是线程安全的（可能是因为它必须在实例变量中保存每个调用者独有的状态），就必须用Prototype方式来配置。

XmlBeanFactory是应用最广泛的BeanFactory接口实现，尤其是对与bean工厂基础上衍生来的应用上下文。

Spring还可以用读取属性文件或者以编程的方式声明bean。对于properties文件可以使用PropertiesBeanDefinitionReader来装载。

编程的方式声明bean，使用RootBeanDefinition和MutablePropertyValues类来协作。

声明bean：

```
DefaultListableBeanFactory bf = new DefaultListableBeanFactory();

MutablePropertyValues pvs1 = new MutablePropertyValues();
pvs1.addPropertyValue("fontName","xxx");
RootBeanDefinition bd1 = new RootBeanDefinition(TextStyle.class,pvs1);
bf.registerBeanDefinition("myStyle",bd1);

MutablePropertyValues pvs2 = new MutablePropertyValues();
pvs2.addPropertyValue("fontName","yyy");
RootBeanDefinition bd2 = new RootBeanDefinition(TextStyle.class,pvs2);
bf.registerBeanDefinition("yourStyle",bd2);
```

上面就是Bean声明，下面是是用Bean：

```
TextStyle myStyle = (TextStyle)bf.getBean("myStyle");
TextStyle yourStyle = (TextStyle)bf.getBean("myStyle");
Map allStyles = bf.getBeanOfTypes(TextStyle.class,false,false);
```

不管Bean声明使用怎样的格式存储，Spring使用者总是可以用一种统一的方式去使用BeanFactory。客户端代码与bean工厂内部的实现完全隔离。只需要通过BeanFactory和ListableBeanFactory接口去查找bean组件。

### 自动装配和依赖检查
bean工厂推荐明确指定组件之间的依赖关系。

Spring的bean工厂还提供了另一种依赖决议的途径：自动装配AutoWired。一个bean被标志为autowired，bean工厂会自动将其他受管对象与其要求的依赖关系进行匹配，从而完成对象的装配。

autowired=“byType”，按照属性的类型自动获得引用。

autowired=“byName”，将对象属性与同名的bean组件关联起来。

推荐使用byType，这不需要精确匹配bean的名称，出错的可能性小。

**依赖检查**会确保bean组件通过JavaBean属性描述的所有依赖关系都得到满足，与自动装配共同使用时特别有用。

dependency-check属性，默认为none，不进行依赖检查：

- objects，表示只做对象间关联的检查。
- simple，只检查基本类型的属性，原始类型或者String。
- all，检查所有的类型。

### 构造器注入
Spring支持setter方法注入，也支持构造器注入。

在bean标签中使用`<constructor-arg>`标签。

与bean属性不同，构造器注入不能按照名称访问，这是Java反射API的限制。如果有多个同一类型的参数，就需要分别为他们指定参数顺序。

Spring的bean工厂也提供了针对构造器的自动装配，由于构造器的参数是不能指定名称的，所以只能通过类型来进行装配。

### 生命周期回调
bean工厂管理的bean除了实例化和属性装配外，通常不需要其他的回调操作。

然而某些对象可能希望获得初始化和销毁处理的回调能力。bean工厂有两种实现方式：

- 实现回调接口，InitializingBean和Disposable。可以在属性设置完成之后和bean工厂销毁之前发送消息给应用对象。
	
    ```
    public interface InitializingBean {
		void afterPropertiesSet() throws Exception;
	}
    ```
    在JavaBean的所有属性设置完成后，容器会调用afterPropertiesSet()方法，应用可在这里执行任何定制的初始化操作。
    
    ```
    public interface DisposableBean {
        void destroy() throws Exception;
    }
    ```
    bean销毁之前可以回调destory方法释放资源。
    
- 在bean组件中声明指定回调方法。方法必须是无参数，public的。

还有一个回调方法可用于访问bean组件所在的bean工厂，只要实现了BeanNameAware接口，在bean组件初始化完成之后，它的setBeanFactory方法就会被调用，从而收到一个bean工厂的引用。

如果bean实现了InitializingBean接口，对setBeanFactory方法的调用会在调用afterPropertiesSet方法之前，如果bean组件声明了初始化方法，对setBeanFactory方法的调用也会在这个初始化方法之前调用。

### 复杂的属性值

## 资源设置
### bean容器中的资源声明
应用对象不需要通过特定的机制查找资源，只需要依赖由外部提供的连接工厂。

## 工厂Bean
为了便于创建任何自定结构的对象，Spring的bean工厂提出了一个特别的概念，工厂bean。它实现了FactoryBean接口。工厂bean本身是在bean工厂中定义的一个bean，同时又是用于创建另一个对象的工厂。工厂bean创建的对象可以被其他bean引用。

FactoryBean接口中getObject方法用于返回被创建的对象，isSingleton方法用来判断getObject方法得到的对象是否是全局唯一的。getObjectType用来获得FactoryBean创建的对象类型。

工厂bean的概念主要用在Spring框架内部。

- JndiObjectFactoryBean，通用工厂bean，通过JNDI查找获得对象。
- LocalSessionFactoryBean，用于在本地装配Hibernate SessionFactory的工厂bean。
- LocalPersistenceManagerFactoryBean，用于在本地装配JDO PersistenceManageFactory的工厂bean。
- TransactionProxyFactoryBean，用于为对象创建事物代理，用于实现简洁易用的声明式事务管理。

还有其他的不在列出。

### 获取JNDI资源
### 创建本地连接工厂

## Spring应用上下文
拥有bean工厂的全部能力之外，应用上下文还具有以下功能：

- 支持不同信息源，可以使用统一的方式获取本地化信息，不依赖于底层的信息来源。默认是从资源绑定中读取信息。
- 访问文件资源。
- 支持应用事件，提供了一个Observer设计模式的实现，可以讲应用事件发布到预先注册的应用监听器上，从而避免了发送者和接受者之间的耦合。监听器还可以获取应用上下文生命周期的事件通知。AOP可以作为事件-监听机制的替代品。

应用上下文的核心接口是ApplicationContext，它继承了ListableBeanFactory，因此应用代码可以吧ApplicationContext当做BeanFactory来用。

应用上下文是应用程序的中心注册机构，允许嵌套，从根上下文开始，可以建立一个应用上下文树。bean的查找首先检查当前的上下文，然后是父上下文，以此类推。

WebApplicationContext继承了ApplicationContext，是一个专门用于web环境的子接口，其中提供了一个方法用于获得当前的ServletContext对象。

现成的ApplicationContext实现有：

- FileSystemXmllApplicationContext
- ClassPathXmlApplicationContext
- XmlWebApplicationContext

### 生命周期回调
应用上下文是bean工厂，就提供了生命周期回调操作，包括InitializingBean和DisposableBean接口、声明性的初始化和销毁方法以及BeanFactoryAware接口等。

bean组件想要获得自己所在的应用上下文，可以实现ApplicationContextAware接口，在初始化的时候回调setApplicationContext方法，将会接收到一个ApplicationContext的引用。

如果bean组件同时还实现了BeanFactoryAware接口，则会首先调用BeanFactoryAware的setBeanFactory方法，然后调用setApplicationContext方法。

Spring内部很多都实现了ApplicationContextAware接口，实现这个接口有两个目的：访问信息源，访问文件资源。

其实还可以实现其他专用接口，比如ResourceLoaderAware接口，不必实现ApplicationContextAware接口。

ServletContextAware接口，可以获得ServletContext对象的引用。

WebApplicationContext接口本身也提供了getServletContext方法来获取ServletContext引用。

### 信息源
除了BeanFactory接口以外，ApplicationContext还继承了MessageSource接口，该接口提供了多个用于获取信息字符串的getMessage方法。

如果一个对象实现了MessageSourceResolvable接口，也可以直接将其传入getMessage方法。

### 文件资源
应用上下文一个重要的特性是可以通用的访问文件资源。ApplicationContext接口继承了ResourceLoader接口，后者的getResource方法可以根据给出的路径返回一个实现了Resource接口的对象，通过它的getInputStream和getFile方法可以统一访问背后的文件资源。

Resource实现：

- ClassPathResource
- FileSystemResource
- ServletContextResource

### Bean Factory后处理
应用上下文允许在底层bean工厂读取bean声明以后进行后处理post-process。最常见的就是bean组件在xml文件中声明，可以对他们进行后处理，覆盖属性的值，或是为某些属性提供占位值。

bean实现BeanFactoryPostProcessor接口即可，当应用上下文启动时，它们会首先被调用，因此可以动态改变其他bean的声明。

现成的实现，PropertyOverrideConfigurer，读取属性文件中beanName.propertyName=value格式的内容，并覆盖bean属性的值。

另一个实现，PropertyPlaceholderConfigurer，可以解析ant风格的占位符。

# 机遇AOP概念的声明性中间件
## AOP
### 定义

- 关注点concern，关注点可以是一个特定的问题、概念、或者是应用程序的兴趣区间。
- 横切关注点crosscutting concern，一个关注点的实现代码，散落在很多个类或方法之中，就称为横切关注点。
- 方面aspect，方面是对一个横切关注点的模块化，将那些散落在各处的用于实现这个关注点的代码规整到一处。
- 连接点join point，程序执行过程中的一点，如：

	+ 方法调用method invocation，对方法的调用。
	+ 字段访问field access，读写实例变量。
	+ 异常抛出throws，特定的异常抛出。

- 增强advice，在特定连接点执行的动作。
- 切入点pointcut，一组连接点的总称，用于指定某个增强应该在何时被调用。
- 引介introduction，为一个现有的Java类或者接口添加方法或字段。
- 混入继承mixin inheritance，一个混入类封装了一组功能，这组功能可以被混入到现有的类中。
- 织入weaving，将方面整合到完整的执行流程中。
- 前增强before pre，在连接点调用之前，首先调用增强。
- 后增强after post，在连接点调用之后，在调用增强。
- 环绕增强around，可以完全控制执行流程。

切入点可分为静态标准切入点和动态标准切入点。静态的根据部署阶段的信息选择增强，比如拦截特定类的所有的getter方法，动态点根据运行时信息选择增强，比如某方法返回值为null，则将其纳入某切入点。

AOP中的常用术语：

- 拦截器interceptor，用来在字段和方法的拦截。
- AOP代理AOP proxy，被增强的对象的引用。
- 目标对象target object，位于拦截器链末端的对象实例。

## AOP实现策略
### 动态代理
动态代理可以使我们为一个或多个接口凭空的创建实现对象，而不需要预先有一个实现类。

好处，Java的语言特性，不需要第三方类库。

缺点，只能针对接口进行代理，不能针对类。

对接口进行代理时，Spring默认使用动态代理。

### 动态字节码生成
针对Java类的代理，可以使用动态字节码生成。CGLIB，Spring中针对类的代理就是使用CGLIB。它可以针对指定的类动态的生成一个子类，覆盖其中的方法，从而实现方法拦截。

CGLIB是通过继承来实现代理，无法为final方法提供代理。

### Java代码生成
生成新的Java源码。

### 使用定制的类加载器
希望对某个类的所有实例进行增强，可以改变new操作符的行为，可以考虑利用Java类加载机制的可扩展性。

### 语言扩展
AspectJ对Java进行了扩展。

## AOP实现

### AspectJ
### AspectWerkz
### JBoss4
### Spring
Spring AOP将AOP支持与Spring轻量级容器基础设施整合在一起，并将Spring提供的服务与AOP组合起来。

Spring AOP功能包括：
- 支持对方法调用进行各种类型的增强，包括环绕增强、前增强、后增强、抛出增强，可以针对接口增强，也可以针对类增强。
- 支持引介和混入，拦截器可以是无状态的，也可以是有状态的，取决于配置的方式。
- 富有表现力并且可扩展的切入点模型，支持正则表达式，支持切入点组合。
- 支持控制流切入点。
- 通过提供适配器类，可以轻松加入任意类型的增强，不必修改核心框架。
- 代理可通过编程驱动，也可以通过配置驱动。
    
### Nanning
### AOP联盟

## Spring中的AOP实践
### 使用ProxyFactoryBean
Spring创建AOP代理的基本途径是使用ProxyFactoryBean。步骤：

- 将必要的切入点、拦截器或其他形式的增强以对象的形式在上下文中声明。
- 在应用上下文中声明一个将要被增强的目标对象。
- 创建一个ProxyFactoryBean实例，它将顶替目标对象被用户使用。



























