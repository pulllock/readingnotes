# JVM类加载器

JVM类加载器有三种：

- 启动类加载器：Bootstrap ClassLoader，负责加载`$JAVA_HOME/jre/lib`目录下的或者被`-Xbootclasspath`参数指定的路径中的类库，C++实现，无法被Java程序直接引用。
- 扩展类加载器：Extension ClassLoader，负责加载`$JAVA_HOME/jre/lib/ext`目录下的或者被`java.ext.dirs`指定的路径中的类库。
- 应用类加载器：Application ClassLoader，负责加载用户类路径（classpath）下的指定的类，可以直接使用该类加载器。

## JVM类加载器加载类的机制

- 当一个类加载器负责加载某个类时，该类所依赖的和引用的其他类也由该类加载器负责载入，除非程序中显式使用另一个类加载器来载入。
- 加载一个类的时候，先让父类加载器进行尝试，如果父类无法加载类，才从当前类加载器的类路径下加载该类。
- 所有加载过的类都会被缓存，类加载器会先从缓存区寻找该类，缓存不存在才会进行加载。

## 类加载的三种方式

- 命令行启动应用时候由JVM初始化加载。
- 通过`Class.forName()`方法动态加载。
- 通过`ClassLoader.loadClass()`方法动态加载。

`Class.forName()`会将class文件加载到JVM中，并且会对类进行解释，执行类中的static块。同时也有个带参数的方法可以控制是否加载static块。只有调用了`newInstance()`方法才会调用构造方法、创建类的对象。

`ClassLoader.loadClass()`只是将class文件加载到JVM中，不会执行static块的内容，只有在`newInstance()`时才会执行static块。

## 双亲委派模型

1. 当AppClassLoader加载一个类时，它会首先把类加载请求委托给父类加载器ExtClassLoader去完成。
2. 当ExtClassLoader加载一个类时，他会首先把类加载请求委托给父类加载器BootstrapClassLoader去完成。
3. 如果BootstrapClassLoader加载失败，会使用ExtClassLoader进行类加载。
4. 如果ExtClassLoader加载失败，会使用AppClassLoader进行类加载。
5. 如果AppClassLoader加载失败，抛出异常`ClassNotFoundException`。

双亲委派模型可以防止内存中出现多份同样的字节码；也可以保证Java的安全。

## 怎么打破双亲委派模型

打破双亲委派模型，需要自己定义一个类加载器，重写loadClass方法，重写findClass方法。双亲委派模型是在loadClass方法中实现的，所以自定义类加载器重写这个方法就可以打破双亲委派模型。

## 打破双亲委派模型的例子

- 使用SPI加载的，如JNDI、JDBC、JCE、JAXB、JBI等等，使用线程上下文类加载器来加载第三方厂商的类。
- tomcat中的web容器类加载器也打破了双亲委派模型，自定义的WebClassLoader除了核心类库外，都优先加载自己路径下的类，加载不到时再交给CommonClassLoader进行双亲委派机制加载。
- OSGI也打破了双亲委派模型，OSGI是网状结构的。
