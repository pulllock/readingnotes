SpringFactoriesLoader和Java的SPI机制一样，SpringFactoriesLoader是Spring框架内部使用的，用来加载Spring一些工厂类的工具。

加载的原理也很简单：

- 使用的时候指定要加载的工厂类型factoryType以及类加载器。
- SpringFactoriesLoader会先根据传入的类加载器到缓存中查一下，如果有就直接返回。
- 缓存中没有就从指定位置`META-INF/spring.factories`加载所有指定的类型的实现类的类名。
- 使用反射实例化所有的实现类，然后返回。

SpringFactoriesLoader在SpringBoot中使用比较多。