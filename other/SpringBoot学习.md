# 构建工具配置
## Maven
可以使用两种方式:继承starter parent或者使用依赖管理器配置.

### 继承spring-boot-starter-parent

```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.3.6.RELEASE</version>
  </parent>
```
接着下面的依赖可以指定导入其他的starter.

### 使用依赖管理
注意加上`<scope>import</scope>`

```
<dependencyManagement>
    <dependencies>
      <dependency>
        <!-- Import dependency management from Spring Boot -->
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>1.3.6.RELEASE</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
```
接着下面的依赖可以指定导入其他的starter.
## Gradle
直接添加各个starter依赖,无需配置parent之类的.

```
dependencies {
    compile("org.springframework.boot:spring-boot-starter-web:1.3.6.RELEASE")
}
```

# spring-boot-starter列表

| 名称 | 描述 |
| ---------- | ---------- |
|spring-boot-starter|核心starter,包含自动配置支持,日志和 YAML配置文件的支持|
|spring-boot-starter-actuator|生产环境,监控和管理应用程序|
|spring-boot-starter-amqp|通过 spring-rabbit 支持 AMQP|
|spring-boot-starter-aop|包含 spring-aop 和 AspectJ 来支持面向切面编程（AOP）|
|spring-boot-starter-artemis|通过Apache Artemis支持JMS API|
|spring-boot-starter-batch|支持Spring Batch包括HSQLDB|
|spring-boot-starter-cache|支持Spring Cache抽象化|
|spring-boot-starter-cloud-connectors|对Spring Cloud Connectors的支持，简化在云平台下（例如，Cloud Foundry 和Heroku）服务的连接|
|spring-boot-starter-data-elasticsearch|对Elasticsearch搜索和分析引擎的支持，包括spring-data-elasticsearch|
|spring-boot-starter-data-gemfire|对GemFire分布式数据存储的支持，包括spring-data-gemfire|
|spring-boot-starter-data-jpa|包含spring-data-jpa,spring-orm和Hibernate来支持JPA|
|spring-boot-starter-data-mongodb|包含spring-data-mongodb来支持MongoDB|
|spring-boot-starter-data-rest|通过spring-data-rest-webmvc支持以REST方式暴露Spring Data仓库|
|spring-boot-starter-data-solr|包含spring-data-solr支持Apache Solr搜索平台|
| spring-boot-starter-freemarker |支持使用FreeMarker作为模板引擎|
|spring-boot-starter-groovy-templates|支持使用groovy作为模板引擎|
|spring-boot-starter-hateoas|通过spring-hateoas支持基于HATEOAS的RESTful服务|
|spring-boot-starter-hornetq|通过HornetQ支持JMS API|
|spring-boot-starter-integration|支持通用spring-integration模块|
|spring-boot-starter-jdbc|支持JDBC|
|spring-boot-starter-jersey|对Jersey RESTful Web服务框架的支持|
|spring-boot-starter-jta-atomikos|通过Atomikos支持JTA分布式事务|
|spring-boot-starter-jta-bitronix|通过Bitronix支持JTA分布式事务|
|spring-boot-starter-mail|对javax.mail的支持|
|spring-boot-starter-mobile|对spring-mobile的支持|
|spring-boot-starter-mustache|支持使用Mustache作为模板引擎|
|spring-boot-starter-redis|包含spring-redis支持REDIS键值数据存储|
|spring-boot-starter-security|支持spring-security|
|spring-boot-starter-social-facebook|支持spring-social-facebook|
|spring-boot-starter-social-linkedin|支持spring-social-linkedin|
|spring-boot-starter-social-twitter|支持spring-social-twitter|
|spring-boot-starter-test|对常用测试依赖的支持,包括JUnit, Hamcrest和Mockito,还有spring-test模块|
|spring-boot-starter-thymeleaf|对Thymeleaf模板引擎的支持,包括和Spring的集成|
|spring-boot-starter-velocity|支持velocity模板引擎|
|spring-boot-starter-web|对全栈web开发的支持,包括Tomcat和spring-webmvc|
|spring-boot-starter-websocket|支持WebSocket开发支持|
|spring-boot-starter-ws|支持Spring Web Services|

生产准备的starts

| 名称 | 描述 |
| ---------- | ---------- |
|spring-boot-starter-actuator|生产环境,监控和管理应用程序|
|spring-boot-starter-remote-shell|支持远程ssh shell|

可替换spring boot中默认的starters

| 名称 | 描述 |
| ---------- | ---------- |
|spring-boot-starter-jetty|导入Jetty HTTP引擎,作为Tomcat的替代|
|spring-boot-starter-log4j|对Log4J日志系统的支持|
|spring-boot-starter-logging|导入Spring Boot的默认日志系统Logback|
|spring-boot-starter-tomcat|导入Spring Boot的默认HTTP引擎Tomcat|
|spring-boot-starter-undertow|导入Undertow HTTP引擎,作为Tomcat的替代|

注意:其他Starters的支持可参考官方文档说明,[Starters](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-starters/README.adoc)


# 日志记录
##Logback
日志记录两种方式:
1. 在src/main/resources(以Maven项目为例)下面创建logback.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml"/>
    <logger name="org.springframework.web" level="DEBUG"/>
</configuration>
```
2. 在application.properties或者application.yml中配置

application.properties:

```
logging.level.org.springframework.web=DEBUG
```

application.yml:

```
spring:
logging:
    level:
      org.springframework.web: DEBUG
```

##Logback多环境配置(yml)参考

```
spring:
  profiles:
    #可在此处选择环境的配置,dev,prod,test
    #也可以在启动时添加参数-Dspring.profiles.active=dev
    active: dev
---
#dev环境
spring:
  profiles: dev
# 日志,logback配置
logging:
  #日志文件
  file: logs/spring-boot-setup.log
  pattern:
    #控制台输出格式
    console: "%d %-5level %logger : %msg%n"
    #文件输出格式
    file: "%d %-5level [%thread] %logger : %msg%n"
  #日志级别
  level:
    org.springframework.web: DEBUG

---
#prod环境
spring:
  profiles: prod
# 日志,logback配置
logging:
  #日志文件
  file: logs/spring-boot-setup.log
  pattern:
    #控制台无输出
    #文件输出格式
    file: "%d %-5level [%thread] %logger : %msg%n"
  #日志级别
  level:
    org.springframework.web: WARN
---
#test环境
spring:
  profiles: test
# 日志,logback配置
logging:
  #日志文件
  file: logs/spring-boot-setup.log
  pattern:
    #控制台输出格式
    console: "%d %-5level %logger : %msg%n"
    #文件输出格式
    file: "%d %-5level [%thread] %logger : %msg%n"
  #日志级别
  level:
    org.springframework.web: INFO
```

# 数据库
## 数据源
以mysql为例:

注意:使用数据库需要在pom文件中添加spring-boot-starter-jdbc和mysql-connector-java的依赖

```
#数据源
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/xxx-test
    username: root
    password: 123456
```

## 数据库连接池

### 默认Tomcat JDBC连接池
Spring Boot默认采用Tomcat JDBC连接池

```
datasource:
    max-idle: 10
    max-wait: 10000
    min-idle: 5
    initial-size: 5
    validation-query: SELECT 1
    test-on-borrow: false
    test-while-idle: true
    time-between-eviction-runs-millis: 18800
    jdbc-interceptors: ConnectionState;SlowQueryReport(threshold=0)
```
## Druid
首先添加上druid的依赖:

```
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>druid</artifactId>
  <version>1.0.18</version>
</dependency>
```

使用其他的连接池,需要配置自己的DataSource bean:

```
@Component
@ConfigurationProperties(prefix = "spring.datasource")
public class DruidConfig {
    private String url;

    private String username;

    private String password;

    private String driverClassName;

    private int initialSize;

    private int maxActive;

    private int minIdle;

    private int maxWait;

    private long timeBetweenEvictionRunsMillis;

    private long minEvictableIdleTimeMillis;

    private String validationQuery;

    private boolean testWhileIdle;

    private boolean testOnBorrow;

    private boolean testOnReturn;

    private boolean poolPreparedStatements;

    private int maxPoolPreparedStatementPerConnectionSize;

    private String filters;

    public DruidConfig() {
    }

    public DruidConfig(String url, String username, String password, String driverClassName, int initialSize, int maxActive, int minIdle, int maxWait, long timeBetweenEvictionRunsMillis, long minEvictableIdleTimeMillis, String validationQuery, boolean testWhileIdle, boolean testOnBorrow, boolean testOnReturn, boolean poolPreparedStatements, int maxPoolPreparedStatementPerConnectionSize, String filters) {
        this.url = url;
        this.username = username;
        this.password = password;
        this.driverClassName = driverClassName;
        this.initialSize = initialSize;
        this.maxActive = maxActive;
        this.minIdle = minIdle;
        this.maxWait = maxWait;
        this.timeBetweenEvictionRunsMillis = timeBetweenEvictionRunsMillis;
        this.minEvictableIdleTimeMillis = minEvictableIdleTimeMillis;
        this.validationQuery = validationQuery;
        this.testWhileIdle = testWhileIdle;
        this.testOnBorrow = testOnBorrow;
        this.testOnReturn = testOnReturn;
        this.poolPreparedStatements = poolPreparedStatements;
        this.maxPoolPreparedStatementPerConnectionSize = maxPoolPreparedStatementPerConnectionSize;
        this.filters = filters;
    }

    @Bean
    @Primary
    public DataSource dataSource() throws Exception{
        DruidDataSource druidDataSource = new DruidDataSource();
        druidDataSource.setUrl(this.url);
        druidDataSource.setUsername(this.username);
        druidDataSource.setPassword(this.password);
        druidDataSource.setDriverClassName(this.driverClassName);
        druidDataSource.setInitialSize(this.initialSize);
        druidDataSource.setMaxActive(this.maxActive);
        druidDataSource.setMinIdle(this.minIdle);
        druidDataSource.setMaxWait(this.maxWait);
        druidDataSource.setTimeBetweenEvictionRunsMillis(this.timeBetweenEvictionRunsMillis);
        druidDataSource.setMinEvictableIdleTimeMillis(this.minEvictableIdleTimeMillis);
        druidDataSource.setValidationQuery(this.validationQuery);
        druidDataSource.setTestWhileIdle(this.testWhileIdle);
        druidDataSource.setTestOnBorrow(this.testOnBorrow);
        druidDataSource.setTestOnReturn(this.testOnReturn);
        druidDataSource.setPoolPreparedStatements(this.poolPreparedStatements);
        druidDataSource.setMaxPoolPreparedStatementPerConnectionSize(this.maxPoolPreparedStatementPerConnectionSize);
        druidDataSource.setFilters(this.filters);
        try {
            if(null != druidDataSource) {
                druidDataSource.setFilters("wall,stat");
                druidDataSource.setUseGlobalDataSourceStat(true);
                druidDataSource.init();
            }
        } catch (Exception e) {
            throw new RuntimeException("load datasource error, dbProperties is :", e);
        }

        return druidDataSource;
    }

 ...
 geter and setter
 ...
```

配置druid的数据监控页面路径和拦截路径:

```
@Bean
    public ServletRegistrationBean druidServlet() {

        return new ServletRegistrationBean(new StatViewServlet(), "/druid/*");
    }

    @Bean
    public FilterRegistrationBean filterRegistrationBean() {
        FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean();
        filterRegistrationBean.setFilter(new WebStatFilter());
        filterRegistrationBean.addUrlPatterns("/*");
        filterRegistrationBean.addInitParameter("exclusions", "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*");
        return filterRegistrationBean;
    }
```
然后浏览器访问http://localhost:8080/druid即可看到界面.

# Mybatis集成
## 添加依赖

```
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.1.1</version>
</dependency>
```

## 配置application.yml

```
#Mybatis配置
mybatis:
  mapperLocations: classpath*:me.cxis.springboot.setup.mapper/*.xml
  typeAliasesPackage: me.cxis.springboot.setup.dto

```

## 编写UserMapper接口和UserMapper.xml文件

UserMapper接口:

```
@Mapper
public interface UserMapper {
    List<User> getUserList();
}
```

UserMapper.xml文件:

```
<mapper namespace="me.cxis.springboot.setup.mapper.UserMapper">
    <select id="getUserList" resultType="me.cxis.springboot.setup.dto.User">
        select * from t_user;
    </select>
</mapper>
```


# 事务管理
在Application中添加注解`@EnableTransactionManagement`启用事务管理,在需要开启事务的地方使用注解`@Transactional`

# FreeMarker模板引擎
## 添加依赖

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-freemarker</artifactId>
</dependency>
```

## 添加配置
在application.yml中添加freemarker配置

```
#freemarker配置
  freemarker:
    cache: false
    charset: UTF-8
    check-template-location: true
    content-type: text/html
    expose-request-attributes: true
    expose-session-attributes: true
    request-context-attribute: request
```

## 创建templates目录
src/main/resources 创建目录 templates,接着在此目录下创建模板文件test.ftl

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Freemarker</title>
</head>
<body>
    Date: ${time?date}
    <br>
    Time: ${time?time}
    <br>
    Message: ${message}
</body>
</html>
```

编写Controller代码:

```
@RequestMapping(value = "test")
    public String testFreeMarker(ModelMap modelMap){
        modelMap.put("time",new Date());
        modelMap.put("message","测试Freemarker");
        return "test";
    }
```

# 集成dubbo
## 添加依赖
分别添加dubbo,zookeeper,zkclient的依赖,同时需要排除依赖中的spring,log4j等

```
<dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>dubbo</artifactId>
        <version>2.5.3</version>
        <exclusions>
          <exclusion>
            <groupId>org.springframework</groupId>
            <artifactId>spring</artifactId>
          </exclusion>
        </exclusions>
      </dependency>
      <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>3.4.6</version>
        <exclusions>
          <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
          </exclusion>
          <exclusion>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
          </exclusion>
        </exclusions>
      </dependency>
      <dependency>
        <groupId>com.github.sgroschupf</groupId>
        <artifactId>zkclient</artifactId>
        <version>0.1</version>
      </dependency>
```
## 服务提供方
添加dubbo.properties文件

```
dubbo.container=log4j,spring
dubbo.application.name=tb-core
dubbo.application.owner=
#dubbo.registry.address=multicast://224.5.6.7:1234
dubbo.registry.address=zookeeper\://127.0.0.1\:2181
#dubbo.registry.address=redis://127.0.0.1:6379
#dubbo.registry.address=dubbo://127.0.0.1:9090
dubbo.monitor.protocol=registry
dubbo.protocol.name=dubbo
dubbo.protocol.port=20881
dubbo.service.loadbalance=roundrobin
dubbo.log4j.file=logs/SpringBootDubboProvider.log
dubbo.log4j.level=DEBUG
```
添加dubbo-provider.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo
	   http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
       <bean id="userService" class="me.cxis.springboot.setup.service.impl.UserServiceImpl"></bean>
       <dubbo:service timeout="3000" retries="0" interface="me.cxis.springboot.setup.service.UserService" ref="userService"/>
</beans>
```

在application添加注解,导入dubbo配置文件

```
@ImportResource("classpath*:dubbo-provider.xml")
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class,args);
    }
}
```

## 服务消费方
添加dubbo.properties文件

```
dubbo.container=log4j,spring
dubbo.application.name=SpringBootDubboConsumer
dubbo.application.owner=
#dubbo.registry.address=multicast://224.5.6.7:1234
dubbo.registry.address=zookeeper\://127.0.0.1\:2181
#dubbo.registry.address=redis://127.0.0.1:6379
#dubbo.registry.address=dubbo://127.0.0.1:9090
dubbo.monitor.protocol=registry
dubbo.protocol.name=dubbo
dubbo.protocol.port=20884
dubbo.service.loadbalance=roundrobin
dubbo.log4j.file=logs/SpringBootDubboConsumer.log
dubbo.log4j.level=DEBUG
```
添加dubbo-consumer.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
	xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
	xsi:schemaLocation="
	http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd  
	http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

	<dubbo:reference id="userService" interface="me.cxis.springboot.setup.service.UserService"/>
</beans>
```

在application添加注解,导入dubbo配置文件

```
@ImportResource("classpath*:dubbo-consumer.xml")
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class,args);
    }
}
```

# 参考
[http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle](http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle)

[https://www.ibm.com/developerworks/cn/java/j-lo-spring-boot/](https://www.ibm.com/developerworks/cn/java/j-lo-spring-boot/)

[https://qbgbook.gitbooks.io/spring-boot-reference-guide-zh/content/](https://qbgbook.gitbooks.io/spring-boot-reference-guide-zh/content/)

[https://springframework.guru/using-yaml-in-spring-boot-to-configure-logback/](https://springframework.guru/using-yaml-in-spring-boot-to-configure-logback/)

[http://blog.csdn.net/catoop/article/details/50501714](http://blog.csdn.net/catoop/article/details/50501714)

[http://my.oschina.net/angerbaby/blog/552936](http://my.oschina.net/angerbaby/blog/552936)

[http://www.voidcn.com/blog/yingxiake/article/p-5930835.html](http://www.voidcn.com/blog/yingxiake/article/p-5930835.html)

