Spring MVC 4的文档阅读。

# Spring MVC

## Spring Web MVC框架简介
Spring MVC围绕DispatcherServlet来设计，这个Servlet会把请求分发给各个处理器。

支持可配置的处理器映射，视图渲染，本地化，时区，主题渲染，文件上传等。

处理器是注解了`@Controller`和`@RequestMapping`的类和方法。

## DispatcherServlet
请求驱动。

所有的设计围绕着一个中央的Servlet展开，它负责把所有请求分发到控制器，同时提供其他web应用开发所需要的功能。DispatcherServlet与IOC容器无缝集成。

DispatcherServlet使用前端控制器的设计模式：

![](DispatcherServlet.png)

DispatcherServlet其实就是Servlet，继承自HttpServlet，需要在web.xml中声明。需要在web.xml中把希望DispatcherServlet处理的请求映射到对应的URL上。

```
<web-app>
	<servlet>
		<servlet-name>example</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<load-on-startup>1</load-on-startup>
	</servlet>
	
	<servlet-mapping>
		<servlet-name>example</servlet-name>
		<url-pattern>/example/*</url-pattern>
	</servlet-mapping>

</web-app>
```
所有路径以`/example`开头的请求都会被名字为example的DispatcherServlet处理。

Servlet3.0+的环境下，还可以使用编程的方式配置Servlet容器。与web.xml配置文件是等效的。

```
public class MyWebApplicationInitializer implements WebApplicationInitializer {

	@Override
	public void onStartup(ServletContext container) {
		ServletRegistration.Dynamic registration = container.addServlet("dispatcher",new DispatcherServlet());
		registration.setLoadOnStartup(1);
		registration.addMapping("/example/*");
		
	}

}
```

WebApplicationInitializer是Spring MVC提供的一个接口，会查找所有基于代码的配置，并应用他们来初始化Servlet3版本以上的web容器。有一个抽象的实现AbstractDispatcherServletInitializer，用于简化DispatcherServlet的注册工作，只需要指定servlet映射即可。

接下来还需要配置除DispatcherServlet以外的其他的bean。

Spring中的ApplicationContext实例是可以有范围的，在Spring MVC中每个DispatcherServlet都持有一个自己的上下文对象WebApplicationContext，它又继承了根WebApplicationContext对象中已经定义的所有bean。这些继承的bean可以在具体的Servlet实例中被重载，在每个Servlet实例中你也可以定义其Scope下的新bean。

DispatcherServlet初始化过程中，SpringMVC会在web的WEB-INF目录下找一个名为`[servlet-name]-servlet.xml`的配置文件，并创建其中定义的bean。如果全局上下文中存在同名bean，会被覆盖。

当应用中只需要一个DispatcherServlet时，只配置一个根context对象也是可以的，只要在servlet初始化参数中配置一个空的contextConfigLocation就可以。

```
<web-app>
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>/WEB-INF/root-context.xml</param-value>
	</context-param>
	<servlet>
		<servlet-name>dispatcher</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class
		>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value></param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>
	
	<servlet-mapping>
		<servlet-name>dispatcher</servlet-name>
		<url-pattern>/*</url-pattern>
	</servlet-mapping>
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listene
	r-class>
	</listener>

</web-app>
```

WebApplicationContext继承自ApplicationContext，提供了一些web常用的特性。与普通的ApplicationContext不同在于，支持主题的解析，知道关联到的是哪个servlet，持有一个该servletContext的引用。

WebApplicationContext被绑定在ServletContext中，如果需要获取可通过RequestContextUtils类中的方法拿到web应用上下文的WebApplicationContext。

### WebApplicationContext中特殊的bean类型