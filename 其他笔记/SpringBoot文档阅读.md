# 安装Spring Boot
## 使用Maven
pom文件需要继承`spring-boot-starter-parent`，依赖声明只需要添加`starters`。

pom.xml:

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>me.cxis</groupId>
    <artifactId>test.springboot</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <!-- 继承spring-boot-starter-parent -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.0.BUILD-SNAPSHOT</version>
    </parent>

    <!-- 添加需要的依赖，starter-web是web项目所需的依赖 -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
</project>
```

## 使用Gradle
build.gradle：

```
buildscript {
    repositories {
        jcenter()
        maven { url 'http://repo.spring.io/snapshot' }
        maven { url 'http://repo.spring.io/milestone' }
    }
    dependencies {
        classpath 'org.springframework.boot:spring-boot-gradle-plugin:2.0.0.BUILD-SNAPSHOT'
    }
}

apply plugin: 'java'
apply plugin: 'org.springframework.boot'

jar {
    baseName = 'test.springboot'
    version =  '0.0.1-SNAPSHOT'
}

//仓库地址
repositories {
    jcenter()
    maven { url "http://repo.spring.io/snapshot" }
    maven { url "http://repo.spring.io/milestone" }
}

//所需的依赖
dependencies {
    compile("org.springframework.boot:spring-boot-starter-web")
    testCompile("org.springframework.boot:spring-boot-starter-test")
}
```

# 开发第一个Spring Boot应用
## 创建pom文件

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>me.cxis</groupId>
    <artifactId>test.first.springboot</artifactId>
    <version>0.0.1-SNAPSHOT</version>

	<!--继承spring-boot-starter-parent-->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.0.BUILD-SNAPSHOT</version>
    </parent>

</project>
```

## 添加依赖
web项目需要添加spring-boot-starter-web依赖：

```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

## 编写代码

```
package me.cxis.first;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * Created by cheng.xi on 28/12/2016.
 */

@RestController
@EnableAutoConfiguration
public class Example {
    
    @RequestMapping("/")
    public String home(){
        return "Hello Spring Boot...";
    }

    public static void main(String[] args) {
        SpringApplication.run(Example.class,args);
    }
}
```

`@RestController`和`@RequestMapping`都是Spring MVC的注解。

`@EnableAutoConfiguration`注解，是class级别的注解，会自动判断需要什么配置，我们在pom文件中添加了`spring-boot-starter-web`依赖，此依赖会自动添加tomcat和Spring MVC依赖，此时AutoConfiguration会自动认为我们开发的是web应用。

## 运行代码

## 创建可执行的Jar包
需要在pom文件中添加`spring-boot-maven-plugin`插件：

```
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```
运行`mvn package`打包，可以在target目录下找到jar：test.first.springboot-1.0-SNAPSHOT.jar。

执行 `java -jar test.first.springboot-1.0-SNAPSHOT.jar`可以运行该程序。

# 构建系统
Maven或Gradle

## 依赖管理

## Maven
pom继承`spring-boot-starter-parent`之后会有很多默认选项提供：

* 默认编译器为Java1.6
* UTF-8编码
* 继承`spring-boot-dependencies`获得默认的依赖管理
* 资源过滤
* 插件配置
* 自动过滤配置文件，`application.properties`,`application.yml`

默认配置文件使用Sring风格占位符`${...}`，Maven使用`@..@`风格的占位符，可通过Maven的`resource.delimiter`属性进行更改。

### 继承parent starter
代码如下：

```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.0.BUILD-SNAPSHOT</version>
</parent>
```

继承以上的starter之后，其他依赖版本不需要自己指定，Spring已经处理好。如果有需要也可以自己指定，在pom中添加类似代码:

```
<properties>
    <spring-data-releasetrain.version>Fowler-SR2</spring-data-releasetrain.version>
</properties>
```

### 不使用parent POM
可以选择不继承`spring-boot-starter-parent`，使用以下代码：

```
<dependencyManagement>
     <dependencies>
        <dependency>
            <!-- 从Spring Boot导入依赖管理 -->
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>2.0.0.BUILD-SNAPSHOT</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```
这样依然可以使用Spring的依赖管理，但是插件管理就没用了。

如果需要指定某个依赖的版本，需要在`spring-boot-dependencies`之前指定依赖信息：

```
<dependencyManagement>
    <dependencies>
        <!-- 使用自己的版本，需要在spring-boot-dependencies之前定义-->
        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-releasetrain</artifactId>
            <version>Fowler-SR2</version>
            <scope>import</scope>
            <type>pom</type>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>2.0.0.BUILD-SNAPSHOT</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 更改Java版本
`spring-boot-starter-parent`选择使用更保守的Java版本，可以通过以下来更改：

```
<properties>
    <java.version>1.8</java.version>
</properties>
```

### 使用Spring Boot的Maven插件
提供了可以打包成可执行jar的Maven插件

```
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

## Gradle
Gradle可以直接导入starter，没有parent之类的需要导入。

```
repositories {
    maven { url "http://repo.spring.io/snapshot" }
    maven { url "http://repo.spring.io/milestone" }
}

dependencies {
    compile("org.springframework.boot:spring-boot-starter-web:2.0.0.BUILD-SNAPSHOT")
}
```

`spring-boot-gradle-plugin`插件提供创建可执行jar，依赖管理，删除SpringBoot预制的版本号等。

```
buildscript {
    repositories {
        jcenter()
        maven { url 'http://repo.spring.io/snapshot' }
        maven { url 'http://repo.spring.io/milestone' }
    }
    dependencies {
        classpath 'org.springframework.boot:spring-boot-gradle-plugin:2.0.0.BUILD-SNAPSHOT'
    }
}

apply plugin: 'java'
apply plugin: 'org.springframework.boot'

repositories {
    jcenter()
    maven { url 'http://repo.spring.io/snapshot' }
    maven { url 'http://repo.spring.io/milestone' }
}

dependencies {
    compile("org.springframework.boot:spring-boot-starter-web")
    testCompile("org.springframework.boot:spring-boot-starter-test")
}
```

## Ant
跳过了！！！

## Starters说明
一组依赖描述，需要某个功能时，直接添加starter就行了，不需要添加其他的依赖，比如需要Spring和JPA，只需要添加`spring-boot-starter-data-jpa`依赖即可。

Spring Boot提供的包含在`org.springframework.boot`组下的starters：

| 名称 | 描述 |
| ---------- | ---------- |
|spring-boot-starter-thymeleaf|对Thymeleaf模板引擎的支持,包括和Spring的集成|
|spring-boot-starter-data-couchbase|对Couchbase数据库的支持|
|spring-boot-starter-artemis|通过Apache Artemis支持JMS API|
|spring-boot-starter-web-service|支持Spring Web Services|
|spring-boot-starter-mail|对javax.mail的支持|
|spring-boot-starter-redis|包含spring-redis支持REDIS键值数据存储|
|spring-boot-starter-web|对全栈web开发的支持,包括Tomcat和spring-webmvc|
|spring-boot-starter-activemq|通过Apache ActiveMQ支持JMS API|
|spring-boot-starter-data-elasticsearch|对Elasticsearch搜索和分析引擎的支持，包括spring-data-elasticsearch|
|spring-boot-starter-data-integration |对Spring企业集成的支持|
|spring-boot-starter-test|对常用测试依赖的支持,包括JUnit, Hamcrest和Mockito,还有spring-test模块|
|spring-boot-starter-jdbc|支持JDBC|
|spring-boot-starter-mobile |支持使用SpringMobile构建web程序|
|spring-boot-starter-validation |支持使用Hibernate Validator的JavaBean校验|
|spring-boot-starter-hateoas|通过spring-hateoas支持基于HATEOAS的RESTful服务|
|spring-boot-starter-jersey|对Jersey RESTful Web服务框架的支持|
|spring-boot-starter-data-neo4j|对neo4j数据库的支持|
|spring-boot-starter-websocket|支持WebSocket开发支持|
|spring-boot-starter-aop|包含 spring-aop 和 AspectJ 来支持面向切面编程（AOP）|
|spring-boot-starter-amqp|支持 AMQP和Rabbit MQ|
|spring-boot-starter-data-cassandra|支持cassandra分布式数据库|
|spring-boot-starter-social-facebook|支持spring-social-facebook|
|spring-boot-starter-jta-atomikos|通过Atomikos支持JTA分布式事务|
|spring-boot-starter-security|支持spring-security|
|spring-boot-starter-mustache|支持使用Mustache作为模板引擎|
|spring-boot-starter-data-jpa|包含spring-data-jpa,spring-orm和Hibernate来支持JPA|
|spring-boot-starter|核心starter,包含自动配置支持,日志和 YAML配置文件的支持|
|spring-boot-starter-groovy-templates|支持使用groovy作为模板引擎|
|spring-boot-starter-batch|支持Spring Batch包括HSQLDB|
|spring-boot-starter-social-linkedin|支持spring-social-linkedin|
|spring-boot-starter-cache|支持Spring Cache抽象化|
|spring-boot-starter-data-solr|包含spring-data-solr支持Apache Solr搜索平台|
|spring-boot-starter-data-mongodb|包含spring-data-mongodb来支持MongoDB|
|spring-boot-starter-jooq|使用jOOQ访问数据库，可以使用`spring-boot-starter-data-jpa`或`spring-boot-starter-jdbc`代替|
|spring-boot-starter-jta-narayana|通过narayana支持JTA分布式事务|
|spring-boot-starter-cloud-connectors|对Spring Cloud Connectors的支持，简化在云平台下（例如，Cloud Foundry 和Heroku）服务的连接|
|spring-boot-starter-jta-bitronix|通过Bitronix支持JTA分布式事务|
|spring-boot-starter-social-twitter|支持spring-social-twitter|
|spring-boot-starter-data-rest|通过spring-data-rest-webmvc支持以REST方式暴露Spring Data仓库|

生产准备的starts

| 名称 | 描述 |
| ---------- | ---------- |
|spring-boot-starter-actuator|生产环境,监控和管理应用程序|

可替换spring boot中默认的starters

| 名称 | 描述 |
| ---------- | ---------- |
|spring-boot-starter-undertow|导入Undertow HTTP引擎,作为Tomcat的替代|
|spring-boot-starter-jetty|导入Jetty HTTP引擎,作为Tomcat的替代|
|spring-boot-starter-logging|导入Spring Boot的默认日志系统Logback|
|spring-boot-starter-tomcat|导入Spring Boot的默认HTTP引擎Tomcat|
|spring-boot-starter-log4j2|对Log4J2日志系统的支持|

# 结构化代码
## 使用default包
尽量避免使用default包，因为在使用`@ComponentScan`, `@EntityScan`或 `@SpringBootApplication`时候可能会出问题。

## Main class的位置
尽量吧Main class放在根包的位置处，`@EnableAutoConfiguration`注解通常会写在Main class处，它会从根包开始搜索其他配置注解等。

# 配置相关的类
Spring Boot使用基于Java的配置，虽然可以通过`SpringApplication.run()`搭配xml来启动，但是更好的办法是在Main方法所在类使用`@Configuration`注解。

## 导入其他配置类
不需要只在一个类中使用`@Configuration`注解，`@Import`注解可以导入其他的注解类。还可以用`@ComponmentScan`注解来自动扫描包括`@Configuration`在内的所有的组件。

## 导入xml配置
可以使用`@ImportResource`注解来加载xml配置文件。

# 自动配置
根据所添加的jar依赖，Spring Boot会自动推测出项目所需要的配置。比如，如果HSQLDB的jar包存在，并且没有手动配置任何有关数据库连接的bean，就会被自动配置为一个基于内存的数据库。

开启自动配置需要添加`@EnableAutoConfiguration`或`@SpringBootApplication`注解到任意一个`@Configuration`注解的类上。

## 逐渐代替自动配置
可以自定义配置，如果添加了自己的DataSource的bean，默认内嵌的数据库支持将不被启用。

可以使用`--debug`来启动应用，查看哪些是自动配置的。

## 禁用指定的自动配置
如果不想用某些自动配置，可以使用exclude禁用，代码如下：

```
@Configuration
@EnableAutoConfiguration(exclude={DataSourceAutoConfiguration.class})
public class MyConfiguration {
}
```

还可以通过`spring.autoconfigure.exclude`属性来控制。

# Spring bean和依赖注入
我们通常会将`@ComponentScan`和`@Autowired`注解配合使用。如果Main类在根包中的话，使用`@ComponentScan`的时候不需要任何其他参数，`@Component`，`@Service`，`@Repository`，`@Controller`等注解都可以被自动的注册成Spring的Bean。

# @SpringBootApplication注解
通常在main类上使用`@Configuration`，`@EnableAutoConfiguration`和`@ComponentScan`就可以了。

Spring Boot还提供了另外一种选择`@SpringBootApplication`，它就等效于上面三个注解配合使用。

# 运行程序
## 直接从IDE运行
## 打Jar包运行
## 使用Maven插件运行
`mvn spring-boot:run`

##使用Gradle插件运行
`gradle bootRun`

## 热部署
可以考虑使用JRebel或者Spring Loaded来解决。

`spring-boot-devtools`模块提供快速的重启。

# 开发者工具
Spring Boot提供了一些额外的工具集合，`spring-boot-devtool`模块可以被添加进任何的项目中，在开发时候提供额外的特性。

Maven：

```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

Gradle：

```
dependencies {
    compile("org.springframework.boot:spring-boot-devtools")
}
```

## 默认属性
在`application.properties`文件中可以设置缓存选项，比如Thymeleaf模板引擎提供了`spring.thymeleaf.cache`属性。

`spring-boot-devtools`模块会自动禁用这些缓存选项。

## 自动重启
使用了`spring-boot-devtools`之后，文件发生变化之后，会自动重启。

## 排除资源
某些资源文件改变时不需要触发重启。默认情况下，改变`/META-INF/maven`,` /META-INF/resources `,`/resources `,`/static` ,`/public` ,` /templates `等目录下的资源文件不会触发重启，只会触发即时重新加载。

自定义排除资源可以使用`spring.devtools.restart.exclude`属性。

比如排除static和public目录：

```
spring.devtools.restart.exclude=static/**,public/**
```

不排除默认的，而需要排除其他的资源，可以使用`spring.devtools.restart.additional-exclude`属性。

### 监测额外的路径
可以使用`spring.devtools.restart.additional-paths`属性来配置额外路径的监测。

### 禁用重启
禁用重启可以使用`spring.devtools.restart.enabled`属性，可以在`application.properties`文件中更改这个值，但是这样的话有关重启的类仍会被加载，只是不会监测文件的变化。

如果需要彻底关闭，需要在`SpringApplication.run(…​)`之前设置系统属性。代码如下：

```
public static void main(String[] args) {
    System.setProperty("spring.devtools.restart.enabled", "false");
    SpringApplication.run(MyApp.class, args);
}
```

### 使用触发文件
如果想自定义触发时间，可以使用触发文件，使用`spring.devtools.restart.trigger-file`属性来指定使用触发文件。

### 自定义重启类加载器
重启的功能使用两个类加载器实现，一般不会出问题，但有时候可能会引起类加载的问题。

默认情况下IDE打开的项目会被重启的类加载器加载，`.jar`文件被基本类加载器加载。
如果想自定义的话，需要创建`META-INF/spring-devtools.properties`文件。

此文件可以包含以`restart.exclude.`和`restart.include.`为前缀的属性。include包含的东西应该被重启类加载器加载，exclude排除的东西应该被基本类加载器加载。可以使用正则表达式来定义。

### 已知的限制
重启功能对标准的`ObjectInputStream `支持不太好，需要使用`ConfigurableObjectInputStream`和`Thread.currentThread().getContextClassLoader()`来配合使用。

## 实时刷新
`spring-boot-devtools`模块包含一个实时刷新的服务器，当资源文件改变时可以触发浏览器刷新。可以把`spring.devtools.livereload.enabled`属性设为false来禁用实时刷新。

## 全局设置
可以添加一个隐藏文件`.spring-boot-devtools.properties`到`$HOME`目录下，此文件用来配置全局设置。

## 远程应用
Sprig Boot开发者工具还可以用在远程应用上。

此部分省略！！！！！！！

# 打包应用用于生产环境
可执行jar可以用在生产环境。

# SpringApplication
`SpringApplication `类提供启动Spring 应用的便捷方法，应用从main方法启动。只需要委托给静态方法`SpringApplication.run`。

## 启动失败
如果应用启动失败，可以注册`FailureAnalyzers`来显示错误信息，采用混合方法来处理问题。

Spring Boot提供了多个`FailureAnalyzers`的实现，可以很方便的添加自己需要的。

如果没有提供失败分析器去处理启动异常，仍然可以通过启用debug属性或者启动debug日志来显示错误信息。

可以使用`java -jar xxxx.jar --debug`来显示。

## 自定义标志
如果`banner.txt`文件在类路径中，启动时候可以显示指定的标志，还可以使用`banner.location`来指定具体的文件。

`banner.charset`来设置编码，默认为UTF-8。

还可以添加`banner.gif`，`banner.jpg`，`banner.png`。

`banner.image.location`属性来设置图片文件的位置。

`banner.txt`文件可以包含一下占位符：

|变量|描述|
|-----|----------|
|`${application.version}`|版本号，声明在`MANIFEST.MF`中|
|`${application.formatted-version}`|声明在`MANIFEST.MF`中的版本号，可以被格式化输出|
|`${spring-boot.version}`|SpringBoot的版本号|
|`${spring-boot.formatted-version}`|被格式化的版本号|
|`${Ansi.NAME}` (或者 `${AnsiColor.NAME}`, `${AnsiBackground.NAME}`, `${AnsiStyle.NAME}`)|`NAME`是ANSI码|
|`${application.title}`|声明在`MANIFEST.MF`中的应用的名字|

`SpringApplication.setBanner(…​)`也可以设置标志。

`spring.main.banner-mode`设置是否显示在console上或者log或者关闭。

YAML中配置：

```
spring:
    main:
        banner-mode: "off"
```

## 自定义SpringApplication
可以通过`application.properties`文件来配置SpringApplication。

## Fluent builder API

## 应用事件和监听
除了Spring的事件之外，SpringApplication还有其他的应用事件。

有些事件在ApplicationContext创建之前就被触发了，所以不能在@Bean的类上创建监听。可以通过`SpringApplication.addListeners(…​)`或者`SpringApplicationBuilder.listeners(…​)`方法来注册监听。

如果希望这些监听器自动注册，并且不关心application怎么创建的话，可以添加`META-INF/spring.factories`文件，使用`org.springframework.context.ApplicationListener`来指定监听类。

程序运行的时候，应用事件发送顺序：

1. `ApplicationStartingEvent `在启动的时候发送，它会在除了注册监听器和初始化之外的任何操作之前发送。
2. `ApplicationEnvironmentPreparedEvent`会在一个已知的上下文被创建之前使用`Environment`的时候被发送。
3. `ApplicationPreparedEvent`在refresh之前bean definitions加载之后发送。
4. `ApplicationReadyEvent`在refresh之后，准备处理服务请求之前发送。
5. `ApplicationFailedEvent`启动出现异常的时候发送。

## Web环境
`SpringApplication`会尝试替你创建正确类型的`ApplicationContext`。默认情况下，如果你创建的是非web应用，`AnnotationConfigApplicationContext`将会被使用；如果是web应用，`AnnotationConfigEmbeddedWebApplicationContext`将被使用。

可以使用`setWebEnvironment(boolean webEnvironment)`来覆盖默认设置。

如果想完全自己控制`ApplicationContext`类型，可以使用`setApplicationContextClass(…​)`。

当使用`SpringApplication`和Junit做测试的时候，建议使用`setWebEnvironment(false)`，设置为非web环境。

## 访问应用的参数
如果想访问`SpringApplication.run(…​)`传递的参数，只需要注入一个`org.springframework.boot.ApplicationArguments`bean。`ApplicationArguments`接口提供访问原始String[]参数，也解析option和non-option参数：

```
import org.springframework.boot.*
import org.springframework.beans.factory.annotation.*
import org.springframework.stereotype.*

@Component
public class MyBean {

    @Autowired
    public MyBean(ApplicationArguments args) {
        boolean debug = args.containsOption("debug");
        List<String> files = args.getNonOptionArgs();
        // 如果运行程序带有一下参数的话 "--debug logfile.txt" debug=true, files=["logfile.txt"]
    }

}
```

Spring Boot也会在Spring Environment中注册一个`CommandLinePropertySource`，这样就可以使用`@Value`注解注入单个的参数。

## 使用ApplicationRunner或CommandLineRunner
如果想在`SpringApplication`启动之前运行一些代码，可以实现`ApplicationRunner`或`CommandLineRunner`接口，这两个接口都提供了一个run方法，可以在`SpringApplication.run(…​)`完成之前调用。

`CommandLineRunner`接口可以把程序的参数当做一个String数组解析；`ApplicationRunner`使用`ApplicationArguments`接口去处理。

如果有多个`CommandLineRunner`或`ApplicationRunner`存在，并且需要按顺序执行，可以实现`org.springframework.core.Ordered`接口或者使用`org.springframework.core.annotation.Order`注解来指定顺序执行。

## 程序退出
每个SpringApplication都会注册一个关闭钩子到JVM，确保ApplicationContext安全退出。

还可以实现`org.springframework.boot.ExitCodeGenerator`接口，在退出的时候执行指定代码。

## 管理特性
通过`spring.application.admin.enabled`属性开启管理特性，这将会在MBeanServer上启用`SpringApplicationAdminMXBean`，可以远程管理Spring Boot应用。

# 外部配置
可以使用properties文件，YAML文件，系统变量，命令行参数等。使用`@Value`可以直接注入属性值，通过Environment抽象或者通过`@ConfigurationProperties`来访问。

Spring Boot提供了`PropertySource`顺序：

1. 首先是开发工具全局设置的属性（在`~/.spring-boot-devtools.properties`文件中）
2. `@TestPropertySource`注解的属性
3. `@SpringBootTest#properties`注解的属性
4. 命令行参数
5. `SPRING_APPLICATION_JSON`的属性
6. `ServletConfig`初始化参数
7. `ServletContext`初始化参数
8. `java:comp/env`JNDI属性
9. `System.getProperties()`Java系统属性
10. 操作系统属性变量
11. `RandomValuePropertySource`属性
12. 已打包jar文件外面的特定配置`application-{profile}.properties`或YAML中的属性
13. 已打包jar文件内部的特定配置`application-{profile}.properties`或YAML中的属性
14. 已打包jar文件外部的应用属性`application.properties`或YAML中的属性
15. 已打包jar文件内部的应用属性`application.properties`或YAML中的属性
16. 在`@Configuration`类中带有`@PropertySource`注解的属性
17. 默认属性`SpringApplication.setDefaultProperties`

如下代码：

```
import org.springframework.stereotype.*
import org.springframework.beans.factory.annotation.*

@Component
public class MyBean {

    @Value("${name}")
    private String name;

    // ...

}
```
上面的name可以使用application.properties中提供name的值，还可以使用`java -jar xxx.jar --name="SpringBoot"`来提供值。

`SPRING_APPLICATION_JSON`可以在命令行提供：`SPRING_APPLICATION_JSON='{"foo":{"bar":"spam"}}' java -jar xxx.jar`

还可以使用：`java -Dspring.application.json='{"foo":"bar"}' -jar xxx.jar`

或者使用：`java -jar xxx.jar --spring.application.json='{"foo":"bar"}'`

或者使用JNDI：`java:comp/env/spring.application.json.`

## 配置随机值
`RandomValuePropertySource `可注入随机值，可生整形，长整形，uuid，字符串等。代码如下：

```
my.secret=${random.value}
my.number=${random.int}
my.bignumber=${random.long}
my.uuid=${random.uuid}
my.number.less.than.ten=${random.int(10)}
my.number.in.range=${random.int[1024,65536]}
```

## 访问命令行属性
`SpringApplication`会转换命令行参数，即带`--`的参数，并添加到Spring的Environment中。

不想使用命令行参数的话使用：`SpringApplication.setAddCommandLineProperties(false)`。

## 应用属性文件
`SpringApplication`会加载`application.properties`文件中的属性到Spring的Environment中去，以下是加载文件的位置：

1. 当前目录下的`/config`目录
2. 当前目录
3. 类路径下的`/config`
4. 类路径

还可以使用YAML（.yml）文件来代替.properties文件。

如果不想使用默认的`application.properties`名字，可以使用`spring.config.name`指定名字，使用`spring.config.location`指定位置：

```
java -jar myproject.jar --spring.config.name=myproject
```

```
java -jar myproject.jar --spring.config.location=classpath:/default.properties,classpath:/override.properties
```

如果`spring.config.location`中包含目录，此目录应该以`/`结尾，此目录将会和`spring.config.name`提供的名字拼接，确定配置文件的路径

##  Profile-specific属性
除了`application.properties`文件外，还可以定义成`application-{profile}.properties`这样的文件，Environment有一个默认的配置，如果没有其他的，就会加载这个默认的文件`application-default.properties`

## 属性中的占位符

```
app.name=MyApp
app.description=${app.name} is a Spring Boot application
```

## 使用YAML替换Properties
### 加载YAML
`YamlPropertiesFactoryBean`会把YAML加载为`Properties`，`YamlMapFactoryBean`会把YAML加载为`Map`。

```
environments:
    dev:
        url: http://dev.bar.com
        name: Developer Setup
    prod:
        url: http://foo.bar.com
        name: My Cool App
```
上面的会被转换成：

```
environments.dev.url=http://dev.bar.com
environments.dev.name=Developer Setup
environments.prod.url=http://foo.bar.com
environments.prod.name=My Cool App
```

```
my:
   servers:
       - dev.bar.com
       - foo.bar.com
```

上面的会被转换成：

```
my.servers[0]=dev.bar.com
my.servers[1]=foo.bar.com
```

如果希望像Spring的DataBinder功能一样，需要在目标bean中提供一个List或者Set，同时提供setter，例如下面代码会绑定上面的YAML配置：

```
@ConfigurationProperties(prefix="my")
public class Config {

    private List<String> servers = new ArrayList<String>();

    public List<String> getServers() {
        return this.servers;
    }
}
```

### YAML转换成Properties
`YamlPropertySourceLoader `可被用来将YAML当做`PropertySource `，这样就可以使用`@Value`注解来访问YAML属性了。

### YAML多配置
YAML中可以存在多套配置，使用`spring.profiles`来指明是哪种配置。

```
server:
    address: 192.168.1.100
---
spring:
    profiles: development
server:
    address: 127.0.0.1
---
spring:
    profiles: production
server:
    address: 192.168.1.120
```

### YAML缺点
不能通过`@PropertySource`加载，如果需要这样加载，只能使用properties文件。

### 合并YAML列表
通过下面的代码可以得到一个包含MyPojo的List：

```
@ConfigurationProperties("foo")
public class FooProperties {

    private final List<MyPojo> list = new ArrayList<>();

    public List<MyPojo> getList() {
        return this.list;
    }

}
```

```
foo:
  list:
    - name: my name
      description: my description
---
spring:
  profiles: dev
foo:
  list:
    - name: my another name
```
上面的配置文件，不管当dev配置有没有激活，上面的List都只包含一个MyPojo，不会自动合并多个条目。


## 类型安全的配置属性
可使用`@Value("${property}")`来注入配置的属性，但是有时候很不方便，SpringBoot提供了另外一种选择。

```
package com.example;

import java.net.InetAddress;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("foo")
public class FooProperties {

    private boolean enabled;

    private InetAddress remoteAddress;

    private final Security security = new Security();

    public boolean isEnabled() { ... }

    public void setEnabled(boolean enabled) { ... }

    public InetAddress getRemoteAddress() { ... }

    public void setRemoteAddress(InetAddress remoteAddress) { ... }

    public Security getSecurity() { ... }

    public static class Security {

        private String username;

        private String password;

        private List<String> roles = new ArrayList<>(Collections.singleton("USER"));

        public String getUsername() { ... }

        public void setUsername(String username) { ... }

        public String getPassword() { ... }

        public void setPassword(String password) { ... }

        public List<String> getRoles() { ... }

        public void setRoles(List<String> roles) { ... }

    }
}
```
上面的POJO定义了一下的属性：

* `foo.enable` 默认为false
* `foo.remote-address`，可从String强制转换
* `foo.security.username`，内部类security
* `foo.security.password`
* `foo.security.roles`，String类型的集合

通过以下方式启用：

```
@Configuration
@EnableConfigurationProperties(FooProperties.class)
public class MyConfiguration {
}
```
当`@ConfigurationProperties`被注册之后，bean会有一个默认的名字：`<prefix>-<fqn>`，`<prefix>`是在`ConfigurationProperties `中指定的名字，`<fqn>`是bean的权限定名，上面的例子会有一个名字：`foo-com.example.FooProperties`。

上面的FooProperties类对应到YAML文件如下：

```
# application.yml

foo:
    remote-address: 192.168.1.1
    security:
        username: foo
        roles:
          - USER
          - ADMIN
```

要想使用FooProperties，直接注入就行了：

```
@Service
public class MyService {

    private final FooProperties properties;

    @Autowired
    public MyService(FooProperties properties) {
        this.properties = properties;
    }

     //...

    @PostConstruct
    public void openConnection() {
        Server server = new Server(this.properties.getRemoteAddress());
        // ...
    }

}
```

### 松散绑定
比如：`context-path`绑定到 `contextPath`，`PORT`绑定到`port`等。

### 属性转换
配置文件中属性的值绑定到有`@ConfigurationProperties`配置的bean时，Spring会自动强制转换成正确的类型。

可以使用`ConversionService`，`CustomEditorConfigurer`，`Converters`实现自定义。

### @ConfigurationProperties校验
默认使用JSR-303来校验配置：

```
@ConfigurationProperties(prefix="connection")
public class FooProperties {

	//添加不为null校验
    @NotNull
    private InetAddress remoteAddress;

	//校验内部类的属性，需要在这里用@Valid触发校验
    @Valid
    private final Security security = new Security();

    // ... getters and setters

    public static class Security {

        @NotEmpty
        public String username;

        // ... getters and setters

    }

}
```

# 多配置
任何被`@Component`和`@Configuration`注解的，都可以被标记成`@Profile`变成配置。

```
@Configuration
@Profile("production")
public class ProductionConfiguration {
    // ...
}
```

使用`spring.profiles.active`来指定哪个是激活的配置。

application.properties：

```
spring.profiles.active=dev,hsqldb
```

命令行：

```
--spring.profiles.active=dev,hsqldb
```

## 添加激活的配置

`spring.profiles.include`可以添加激活的配置，SpringApplication的`setAdditionalProfiles()`也可以。

# 日志
`java -jar myapp.jar --debug`显示为debug模式，或者在`application.properties`中添加`debug=true`。

## 文件输出
默认日志只会输出到终端上，不会输出到文件。如果想输出到文件，需要在`application.properties`中指定`logging.file`或`logging.path`属性。

## 日志级别
通常会使用：`logging.level.* = LEVEL`。

```
logging.level.root=WARN
logging.level.org.springframework.web=DEBUG
logging.level.org.hibernate=ERROR
```

## 自定义日志配置
|日志系统|配置|
|------|----------|
|Logback|`logback-spring.xml`，`logback-spring.groovy`，`logback.xml`，`logback.groovy`|
|Log4j2|`log4j2-spring.xml`，`log4j2.xml`|
|JDK (Java Util Logging)|`logging.properties`|

## Logback扩展
省略！！！

# 开发web应用
使用`spring-boot-starter-web`模块。

## Spring Web MVC framework
### Spring MVC自动配置
在Spring默认的基础上，自动配置添加了以下特性：

* 加入了`ContentNegotiatingViewResolver`和`BeanNameViewResolver`bean
* 支持处理静态资源，包括WebJars
* 自动注册`Converter`，`GenericConverter`，`Formatter`等bean
* 支持`HttpMessageConverters`
* 自动注册`MessageCodesResolver`
* 静态`index.html`的支持
* 支持自定义`Favicon`
* 自动使用`ConfigurableWebBindingInitializer`bean

如果想扩展MVC的配置，比如拦截器，格式化，试图控制器等，只需要添加`WebMvcConfigurerAdapter`类型的`@Configuration`类，但是不需要添加`@EnableWebMvc`。

还可以声明`WebMvcRegistrationsAdapter`实例来提供`RequestMappingHandlerMapping`, `RequestMappingHandlerAdapter`, `ExceptionHandlerExceptionResolver`组件。

如果想完全控制Spring MVC，需要自己使用`@Configuration`和`@EnableWebMvc`注解。

### HttpMessageConverters
Spring MVC使用`HttpMessageConverter`接口转换HTTP的request和response，Objects可以被自动转换成JSON（使用Jackson库），或者转成XML（使用Jsckson或者JAXB）。String默认编码UTF-8。

如果想添加或者自定义转换器，可以使用`HttpMessageConverters`类：

```
import org.springframework.boot.autoconfigure.web.HttpMessageConverters;
import org.springframework.context.annotation.*;
import org.springframework.http.converter.*;

@Configuration
public class MyConfiguration {

    @Bean
    public HttpMessageConverters customConverters() {
        HttpMessageConverter<?> additional = ...
        HttpMessageConverter<?> another = ...
        return new HttpMessageConverters(additional, another);
    }

}
```

### 自定义JSON序列化和反序列化
如果使用Jackson处理JSON数据，想使用自己的JsonSerializer和JsonDeserializer类，通常是通过Jackson的模块去做，但是SpringBoot提供了`@JsonComponent`注解来直接注入Bean。

`@JsonComponent`可以直接使用在JsonSerializer或JsonDeserializer的实现上，也可以用在包含两者的内部类的类上：

```
import java.io.*;
import com.fasterxml.jackson.core.*;
import com.fasterxml.jackson.databind.*;
import org.springframework.boot.jackson.*;

@JsonComponent
public class Example {

    public static class Serializer extends JsonSerializer<SomeObject> {
        // ...
    }

    public static class Deserializer extends JsonDeserializer<SomeObject> {
        // ...
    }

}
```

所有的ApplicationContext中的`@JsonComponent`bean都会被使用Jackson自动注册。

### MessageCodesResolver

### 静态内容
默认处理静态内容的目录：`/static`，`/public`，`/resources`，`/META-INF/resources`。使用SpringMVC的`ResourceHttpRequestHandler`来处理，想要修改默认行为需要添加自己的`WebMvcConfigurerAdapter`，然后重写`addResourceHandlers`方法。

默认处理`/**`的资源，可以修改`spring.mvc.static-path-pattern`来改变位置。

`spring.resources.static-locations`修改静态资源的位置。

`/webjars/**`处理Webjars。

程序需要打包成jar的话，不要使用`src/main/webapp`目录，这目录通常在打包成war包的时候使用，对于很多构建工具来说打jar包的时候会忽略此目录。

### 模板引擎
自动配置支持FreeMarker，Groovy，Thymeleaf，Mustache。如果可能的话，应该避免使用JSP。

默认目录：`src/main/resources/templates`

### 错误处理
对错误处理有个默认的`/error`映射。想要替换默认的错误处理行为，可以自己实现`ErrorController`来完全替换默认的错误处理。使用`ErrorAttributes`只是替换错误的内容。

自定义错误页面：

404:

```
src/
 +- main/
     +- java/
     |   + <source code>
     +- resources/
         +- public/
             +- error/
             |   +- 404.html
             +- <other public assets>
```

FreeMarker模板映射所有的5xx错误：

```
src/
 +- main/
     +- java/
     |   + <source code>
     +- resources/
         +- templates/
             +- error/
             |   +- 5xx.ftl
             +- <other templates>
```

更复杂的应射，自己实现`ErrorViewResolver`接口：

```
public class MyErrorViewResolver implements ErrorViewResolver {

    @Override
    public ModelAndView resolveErrorView(HttpServletRequest request,
            HttpStatus status, Map<String, Object> model) {
        // Use the request or status to optionally return a ModelAndView
        return ...
    }

}
```

还可以继续使用SpringMVC特性，像`@ExceptionHandler`方法和`@ControllerAdvice`。

### CORS跨域支持
在controller方法上使用`@CrossOrigin`注解。

全局跨域配置可以注册`WebMvcConfigurer`bean，自定义`addCorsMappings(CorsRegistry)`方法即可。

```
@Configuration
public class MyConfiguration {

    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurerAdapter() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**");
            }
        };
    }
}
```

## JAX-RS和Jersey
省略！！！

## 嵌入式servlet容器支持
Tomcat，Jetty，Undertow。

### 自定义嵌入式servlet容器
在`application.properties`文件中修改：

* `server.port`端口，`server.address`地址
* session设置`server.session.persistence`，`server.session.timeout`，`server.session.store-dir`，`server.session.cookie.*`
* 错误页面`server.error.path`
* SSL
* Http压缩

还可以实现`EmbeddedServletContainerCustomizer `接口来实现：

```
import org.springframework.boot.context.embedded.*;
import org.springframework.stereotype.Component;

@Component
public class CustomizationBean implements EmbeddedServletContainerCustomizer {

    @Override
    public void customize(ConfigurableEmbeddedServletContainer container) {
        container.setPort(9000);
    }

}
```

直接使用`TomcatEmbeddedServletContainerFactory`, `JettyEmbeddedServletContainerFactory` 或`UndertowEmbeddedServletContainerFactory` 来定义：

```
@Bean
public EmbeddedServletContainerFactory servletContainer() {
    TomcatEmbeddedServletContainerFactory factory = new TomcatEmbeddedServletContainerFactory();
    factory.setPort(9000);
    factory.setSessionTimeout(10, TimeUnit.MINUTES);
    factory.addErrorPages(new ErrorPage(HttpStatus.NOT_FOUND, "/notfound.html"));
    return factory;
}
```

# Security
省略！！！！！！


# Sql数据库
使用`spring.datasource.*`做配置

```
spring.datasource.url=jdbc:mysql://localhost/test
spring.datasource.username=dbuser
spring.datasource.password=dbpass
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```
