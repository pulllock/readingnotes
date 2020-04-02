如果要了解Spring中@Enable的实现机制，需要提前了解下@Import的用途和实现方式，每个@Enable注解上都会有@Import注解，@Import中导入自定的ImportSelector或者ImportBeanDefinitionRegistrar或者配置，这里面会实现相关功能会引入相关功能。

举个例子：@EnableCaching：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(CachingConfigurationSelector.class)
public @interface EnableCaching {

	boolean proxyTargetClass() default false;

	AdviceMode mode() default AdviceMode.PROXY;

	int order() default Ordered.LOWEST_PRECEDENCE;

}

```

@EnableCaching注解上有注解@Import({CachingConfigurationSelector.class})，在处理@Import的时候，会调用CachingConfigurationSelector的selectImports方法来加入缓存相关功能和配置。

可以认为@Enable就是帮我们使用@Import导入一些配置和实现等，我们也可以自己直接使用@Import，不使用@Enable。