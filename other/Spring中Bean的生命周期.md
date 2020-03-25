1. 手动或者自动的触发获取一个Bean，使用BeanFactory的时候需要我们代码自己获取Bean，ApplicationContext则是在IOC启动的时候自动初始化一个Bean。
2. IOC会根据BeanDefinition来实例化这个Bean，如果这个Bean还有依赖其他的Bean则会先初始化依赖的Bean，这里又涉及到了循环依赖的解决。
3. 实例化之前先看下Bean是否实现了InstantiationAwareBeanPostProcessor接口，如果实现了该接口，则先调用其postProcessBeforeInstantiation方法，常见的实现是在AbstractAutoCreator中有该方法的实现，在这里可以根据实际情况返回一个代理，如果这里返回了一个代理Bean而不是返回null的话，下面的步骤就不会继续进行下去了。换句话说，如果这里生成了Bean就不需要下面的步骤进行Bean的实例化和初始化了。
4. 如果上面一步返回了一个Bean，虽然不需要继续下面的实例化步骤，但是还是需要走一下postProcessBeforeInstantiation的postProcessAfterInitialization方法，就是不管怎么样生成Bean，最后都要走一下初始化的后置处理器。
5. 如果上面没有生成一个Bean，就继续正常的实例化Bean，实例化Bean的时候根据工厂方法、构造方法或者简单初始化等选择具体的实例来进行实例化，最终都是使用反射进行实例化。
6. Bean实例化完成，也就是一个对象实例化完成后，如果改Bean实现了MergedBeanDefinitionPostProcessor接口，则调用这个接口的postProcessMergedBeanDefinition方法，这里我们熟悉的有：对`@Autowired, @Value, @Inject`等注解进行预解析；对`@Resource, @PostConstruct, @PreDestroy`等注解进行预解析。
7. 经过对一些注解的预解析之后，会进行循环依赖的一些准备工作，如果需要提前曝光的话，会把还在生成的Bean放到singletonFactories中去，解决循环依赖的时候会用到。
8. 然后会继续填充这个Bean的各个属性，在填充属性前，还需要看是否实现了InstantiationAwareBeanPostProcessor接口，如果实现了该接口，则需要调用其postProcessAfterInstantiation方法，如果有一个实现的该方法返回了false，表示不需要继续再进行下面的填充属性设置，也不需要再继续处理其他的InstantiationAwareBeanPostProcessor。
9. 如果上一步的处理没有直接返回false，而是继续处理的话，还是会继续判断有没有实现InstantiationAwareBeanPostProcessor接口，如果实现了该接口则需要调用其postProcessPropertyValues方法，这里会有我们熟悉的动作：对`@Autowired, @Value, @Inject`等注解的依赖进行设值、对`@Resource, @PostConstruct, @PreDestroy`等注解的依赖进行设值，都是通过反射将依赖设置到目标Bean中去；还有一个对`@Required`所注解的必须依赖进行校验，如果没有就会抛异常。
10. 接下来如果有depends-on属性的话，需要进行依赖检查，不满足依赖会抛异常。
11. 这里就是真正填充属性的地方了，使用反射机制将属性设置到Bean中去。
12. 填充完属性后，会调用各种Aware方法，将需要的组件设置到当前Bean中。BeanFactory这种低级容器需要我们手动注册Aware接口，而ApplicationContext这种高级容器在IOC启动的时候就自动给我们注册了Aware等接口。
13. 接下来如果Bean实现了PostProcessor一系列的接口，会先调用其中的postProcessBeforeInitialization方法。BeanFactory这种低级容器需要我们手动注册PostProcessor接口，而ApplicationContext这种高级容器在IOC启动的时候就自动给我们注册了PostProcessor等接口。实现类大概如下：ApplicationContextAwareProcessor在该方法中会执行各种ApplicationContext的Aware方法，比如ApplicationContextAware、ResourceLoaderAware等；InitDestroyAnnotationBeanPostProcessor在该方法中会调用注解的init方法，也就是`@PostConstruct`注解的方法，比下面的InitializingBean和xml中的init-method执行要早。
14. 如果Bean实现了InitializingBean接口，则会调用对应的afterPropertiesSet方法。
15. 如果Bean设置了init-method属性，则会调用init-method指定的方法，使用反射调用。
16. 接下来如果Bean实现了PostProcessor一系列的接口，会先调用其中的postProcessAfterInitialization方法。BeanFactory这种低级容器需要我们手动注册PostProcessor接口，而ApplicationContext这种高级容器在IOC启动的时候就自动给我们注册了PostProcessor等接口。实现类大概如下：AbstractAutoProxyCreator会在该方法看是否需要进行代理；
17. 接下来继续注册DisposableBean。
18. 到这里Bean就可以使用了。
19. 容器关闭的时候需要销毁Bean。
20. 如果Bean实现了DisposableBean，则调用destroy方法。
21. 如果Bean配置了destroy-method属性，则调用指定的destroy-method方法。

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