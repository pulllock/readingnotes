tomcat中也有很多的自定义的类加载器，保证容器的不同部分，以及运行在容器中的web应用可以访问不同的保存着类和资源的仓库。tomcat的类加载器机制跟jdk的类加载器机制基本类似，但是web应用类加载器处理请求的时候会稍微有些不同，jdk的类加载机制不再重复。

# tomcat类加载器结构
tomcat的类加载器的结构大概如下：

```
        Bootstrap
             |
          System
             |
          Common
          /     \
 Webapp1   Webapp2 ...
```

- Bootstrap，包含JVM的基本运行时类，以及系统扩展目录（`$JAVA_HOME/jre/lib/ext`）下的jar包。
- System，通常是根据CLASSPATH环境变量内容进行初始化的。这些类对tomcat内部类以及web应用都是可见的。tomcat启动脚本`$CATALINA_HOME/bin/catalina.sh`忽略了CLASSPATH的内容，会从以下位置中构建类加载器：
    + `$CATALINA_HOME/bin/bootstrap.jar`包含用来初始化tomcat服务器的main方法，以及它依赖的类加载器实现类。
    + `$CATALINA_BASE/bin/tomcat-juli.jar`或者`$CATALINA_HOME/bin/tomcat-juli.jar`是日志实现类，优先使用`CATALINA_BASE`下的`tomcat-juli.jar`。
    + `$CATALINA_HOME/bin/commons-daemon.jar`引用自bootstrap.jar的清单文件中。
- Common，这种类加载器包含额外的类，对于tomcat内部以及所有web应用都是可见的。一般情况，应用类不会放在这里，Common类加载器会加载`${catalina.base}/lib,${catalina.base}/lib/*.jar,${catalina.home}/lib,${catalina.home}/lib/*.jar`这些目录下的资源或者jar包，该配置在`$CATALINA_BASE/conf/catalina.properties`的`common.loader`属性中。
- Webapp1，Webapp2等，为每个部署在tomcat中的web应用创建的类加载器，每个web应用只能看到自己的`/WEB-INF/classes`和`/WEB-INF/lib`目录，其他应用则不可见。

# web应用加载类顺序

tomcat启动之后，在web应用类加载器处理请求的时候，Web应用类加载器和JVM的类加载器不太一样，不是双亲委派模型。当请求从Web应用的类加载器加载类时，首先查看自己的仓库，而不是委托给父类加载。类加载器会显式的忽略所有包含Servlert API类的Jar文件。tomcat其他的类加载器则使用双亲委派模型。


Web应用类加载器加载顺序为：

- JVM的BootStrap类
- WEB应用的`/WEB-INF/classes`下的类
- WEB应用的`/WEB-INF/lib/*.jar`下的jar
- System类加载器的类
- Common类加载器的类

但是如果web应用类加载器有配置了`<Loader delegate="true"/>`，这时候就会按照双亲委派模型，顺序变为：

- JVM的BootStrap类
- System类加载器的类
- Common类加载器的类
- WEB应用的`/WEB-INF/classes`下的类
- WEB应用的`/WEB-INF/lib/*.jar`下的jar

# tomcat启动时类加载器的创建顺序

tomcat启动的时候，会创建如下的类加载器：

- Bootstrap类加载器，用来加载JVM启动所需要的类，也会加载JVM的标准扩展类
- System类加载器，加载tomcat启动类，比如bootstrap.ja和tomcat-juli.jar。
- Common类加载器，是tomcat的类加载器，加载tomcat自身使用的类，也加载web应用通用的类。
- WebappClassLoader，web应用类加载器，每个应用都会有一个类加载器，会加载自己的`/WEB-INF/classes`和`/WEB-INF/lib/*.jar`下的class和jar文件。

# 其他的类加载器
除了上面说的Common和Webapp类加载器之外，tomcat中还有Server类加载器和Shared类加载器。

- Server类加载器负责加载位于`$CATALINE_HOME/server`目录下的tomcat的核心类，在启动时创建，其父类加载器是Common类加载器。
- Shared类加载器负责加载webapp公用的类，其父类加载器是Common类加载器，也是在tomcat启动时创建。

有关更多的内容会在源码分析里面继续解析，这里解析的并不详细，最重要的是还不太熟悉和了解。
