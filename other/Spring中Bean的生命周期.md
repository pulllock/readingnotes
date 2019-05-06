BeanFactory和ApplicationContext是Spring中两种很重要的容器，前者提供了最基本的依赖注入的支持，后者在继承前者的基础上进行了功能的拓展，增加了事件传播，资源访问，国际化的支持等功能。同时两者的生命周期也稍微有些不同。

# BeanFactory中Bean的生命周期

1. 容器寻找Bean的定义信息，并将其实例化。
2. 使用依赖注入，Spring按照Bean定义信息配置Bean的所有属性。
3. 如果Bean实现了BeanNameAware接口，工厂调用Bean的setBeanName()方法传递Bean的id。
4. 如果实现了BeanFactoryAware接口，工厂调用setBeanFactory()方法传入工厂自身。
5. 如果BeanPostProcessor和Bean关联，那么它们的postProcessBeforeInitialization()方法将被调用。（需要手动进行注册！）
6. 如果Bean实现了InitializingBean接口，则会回调该接口的afterPropertiesSet()方法。
7. 如果Bean指定了init-method方法，就会调用init-method方法。
8. 如果BeanPostProcessor和Bean关联，那么它的postProcessAfterInitialization()方法将被调用。（需要手动注册！）
9. 现在Bean已经可以使用了。
   1. scope为singleton的Bean缓存在Spring IOC容器中。
   2. scope为prototype的Bean生命周期交给客户端。
10. 销毁。
   1. 如果Bean实现了DisposableBean接口，destory()方法将会被调用。
   2. 如果配置了destory-method方法，就调用这个方法。

# ApplicationContext中Bean的生命周期

1. 容器寻找Bean的定义信息，并将其实例化。会对scope为singleton且非懒加载的bean进行实例化
2. 使用依赖注入，Spring按照Bean定义信息配置Bean的所有属性。
3. 如果Bean实现了BeanNameAware接口，工厂调用Bean的setBeanName()方法传递Bean的id。
4. 如果实现了BeanFactoryAware接口，工厂调用setBeanFactory()方法传入工厂自身。
5. 如果实现了ApplicationContextAware接口，会调用该接口的setApplicationContext()方法，传入该Bean的ApplicationContext，这样该Bean就获得了自己所在的ApplicationContext。
6. 如果Bean实现了BeanPostProcessor接口，则调用postProcessBeforeInitialization()方法。
7. 如果Bean实现了InitializingBean接口，则会回调该接口的afterPropertiesSet()方法。
8. 如果Bean制定了init-method方法，就会调用init-method方法。
9. 如果Bean实现了BeanPostProcessor接口，则调用postProcessAfterInitialization()方法。
10. 现在Bean已经可以使用了。
	1. scope为singleton的Bean缓存在Spring IOC容器中。
   	2. scope为prototype的Bean生命周期交给客户端。
11. 销毁。
	1. 如果Bean实现了DisposableBean接口，destory()方法将会被调用。
   	2. 如果配置了destory-method方法，就调用这个方法。

# 两种容器中的不同之处

1. BeanFactory容器中不会调用ApplicationContext接口的setApplicationContext()方法。
2. BeanFactory中BeanPostProcessor接口的postProcessBeforeInitialzation()方法和postProcessAfterInitialization()方法不会自动调用，必须自己通过代码手动注册。
3. BeanFactory容器启动的时候，不会去实例化所有Bean,包括所有scope为singleton且非懒加载的Bean也是一样，而是在调用的时候去实例化。