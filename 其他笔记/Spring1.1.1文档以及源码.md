# Introduction
## Overview
Spring Core 提供依赖注入功能。基本实现是BeanFacctory。

Spring Context 继承了所有BeanFactory功能，添加了消息，资源绑定，资源加载等功能。

Spring Dao 提供JDBC的抽象。

Spring ORM 提供对ORM的支持。

Spring AOP 提供了对AOP的支持。

Spring Web 提供对web的支持。

Spring Web MVC 对web应用mvc的实现。

# Background information
## IOC/DI

# Beans，BeanFactory and the ApplicationContext
## Introduction
`org.springframework.beans`和`org.springframework.context`的包内容提供了Spring的IOC基础。BeanFactory管理各种bean，提供配置框架和基本功能，ApplicationContext在BeanFactory基础上提供了更多的增强功能，是BeanFactory的超集。

## BeanFactory and BeanDefinitions - the basics
### The BeanFactory
BeanFactory是实际的容器，用来实例化，配置，管理bean。常用的实现XMLBeanFactory。

```
public interface BeanFactory {
	//根据名字获取bean
	Object getBean(String name) throws BeansException;

	//根据名字和指定的类型获取bean
	Object getBean(String name, Class requiredType) throws BeansException;

	//是否包含指定名字的bean
	boolean containsBean(String name);

	//是否是单例bean
	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	//获取别名
	String[] getAliases(String name) throws NoSuchBeanDefinitionException;

}
```

### The BeanDefinition
包含 class，id和name，singleton或者prototype，constructor 参数，bean属性，autowiring模式，dependency 检车模式，initialization方法，destruction方法。

```
public interface BeanDefinition {

	//返回bean的class
	Class getBeanClass();

	//是否是抽象的
	boolean isAbstract();

	//是否是单例
	boolean isSingleton();

	//是否是懒加载的
	boolean isLazyInit();

	//获取属性
	MutablePropertyValues getPropertyValues();

	//获取构造方法参数
	ConstructorArgumentValues getConstructorArgumentValues();

	//获取资源描述
	String getResourceDescription();

}
```

### The bean class
class属性是必须的。

#### Bean creation via constructor
#### Bean creation via static factory method
#### Bean creation via instance factory method

### The bean identifiers ( id and name )
### To singleton or not to singleton

## Properties, collaborators, autowiring and dependency checking
### Setting bean properties and collaborators
setter注入和构造器注入

### Bean properties and constructor arguments detailed

### Method Injection
单例的bean需要注入prototype的bean依赖，单例bean可以实现BeanFactoryAware接口，自己代码中实现。

### Lookup method Injection

#### Arbitrary method replacement

### Using depends-on

### Autowiring collaborators

byName

byType、

constructor

autodetect

### Checking for dependencies

## Customizing the nature of a bean
### Lifecycle interfaces

实现InitializingBean接口，Spring会调用afterPropertiesSet()方法。

实现DisposableBean接口，Spring会调用destory()方法。

Spring会使用BeanPostProcessors来处理以上的标记接口。

#### InitializingBean / init-method
实现了InitializingBean接口，BeanFactory可以在bean的所有的必须的属性都设置了之后调用afterPropertiesSet()方法，来实现额外功能。

#### DisposableBean / destroy-method
实现了DisposableBean之后，当Bean销毁的时候，BeanFactory会调用destory()方法，做其他处理。

### Knowing who you are
#### BeanFactoryAware
实现BeanFactoryAware接口，可以在setBeanFactory方法中获取BeanFactory。

#### BeanNameAware
实现BeanNameAware接口，可以获取到bean的id，回调方法会在正常属性填充之后，在InitializationBean的afterPropertiesSet或者init-method之前调用。

### FactoryBean
FactoryBean需要被工厂类实现。

## Abstract and child bean definitions
ChildBeanDefinition

## Interacting with the BeanFactory

### Obtaining a FactoryBean, not its product

- getBean("myBean")返回我们定义的bean的实例。
- getBean("&myBean")返回FactoryBean实例

## Customizing beans with BeanPostprocessors
BeanPostprocessors的方法会在初始化方法（afterPropertiesSet或者init-method）方法之前调用。

## Customizing bean factories with BeanFactoryPostprocessors
Spring中内置的：PropertyResourceConfigurer，PropertyPlaceHolderConfigurer，BeanNameAutoProxyCreator

### The PropertyPlaceholderConfigurer
### The PropertyOverrideConfigurer
## Registering additional custom PropertyEditors
CustomEditorConfigurer

## Introduction to the ApplicationContext

- MessageSource,
- Access to resources
- Event propagation
- Loading of multiple (hierarchical) contexts

## Added functionality of the ApplicationContext
### Using the MessageSource
### Propagating events
ApplicationEvent和ApplicationListener。

内置的事件：

- ContextRefreshedEvent
- ContextClosedEvent
- RequestHandledEvent

### Using resources within Spring
## Customized behavior in the ApplicationContext
### ApplicationContextAware marker interface
ApplicationContextAware，实现此接口，创建bean的时候会回调setApplicationContext()方法。

### The BeanPostProcessor
### The BeanFactoryPostProcessor
### The PropertyPlaceholderConfigurer

## Registering additional custom PropertyEditors
## Setting a bean property as the result of a method invocation


## Creating an ApplicationContext from a web application

# Spring AOP
## Concepts
### AOP Proxies in Spring
默认使用动态代理来实现AOP代理，只能代理接口。

还可以使用CGLIB代理，可以代理实现类。

## Pointcuts in Spring

Pointcut接口

```
public interface Pointcut {
	//限制切点的目标类
	ClassFilter getClassFilter();
	//方法匹配
	MethodMatcher getMethodMatcher();
	
}
```

#### Static pointcuts




































