# JVM类加载器

JVM类加载器有三种：

- 启动类加载器：Bootstrap ClassLoader，负责加载`$JAVA_HOME/jre/lib`目录下的或者被`-Xbootclasspath`参数指定的路径中的类库，C++实现，无法被Java程序直接引用。
- 扩展类加载器：Extension ClassLoader，负责加载`$JAVA_HOME/jre/lib/ext`目录下的或者被`java.ext.dirs`指定的路径中的类库。