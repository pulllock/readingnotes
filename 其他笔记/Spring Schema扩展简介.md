# 介绍
Spring从2.0开始引入新机制，用于扩展xml模式，可以编写自定义的xml bean解析器，集成到Spring Ioc容器中。

## 步骤
1. classpath下的META-INF中定义两个文件：spring.handlers、spring.schemas，用来识别配置
2. 编写xml xsd文件，来描述自定义元素
3. 编写NamespaceHandler实现类
4. 编写BeanDefinitionParser实现类


# 识别配置
* spring.handlers中指明命名空间需要哪个类来处理。
* spring.schemas指明了schema文件的位置，xsd文件

Spring容器在启动的时候会根据这两个文件的配置加载文件内容，然后解析文件中的配置信息。

# 配置文件


# 解析配置
通过实现NamespaceHandlerSupport，BeanDefinitionPaser对自定义的schema文件完成解析。

NamespaceHandlerSupport会根据schema的节点名称来找到某个BeanDefinitionParser，然后由BeanDefinitionParser来完成具体的解析工作。

