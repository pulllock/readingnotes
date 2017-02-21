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
DispatcherServlet使用了特殊的bean来处理请求，渲染视图等。如果想指定哪个bean，可在web应用上下文WebApplicationContext中简单的配置他们。

SpringMVC维护了一个默认的bean列表，没有进行特别设置的话，框架会选用默认的bean。

HandlerMapping，处理器映射，根据规则将请求映射到具体的处理器以及一系列的前置处理器和后置处理器。

HandlerAdapter，处理器适配器，拿到请求对应的处理器后，适配器将负责去调用该处理器，这使得DispatcherServlet无需关心具体的调用细节。

HandlerExceptionResolver，处理器异常解析器，负责将捕获的异常映射到不同的视图上去。

ViewResolver，视图解析器，负责将一个代表逻辑视图名的字符串映射到实际的视图类型View上。

LocaleResolver，LocaleContextResolver，地区解析器和地区上下文解析器，负责解析客户端所在的地区信息时区信息，为国际化的视图定制提供了支持。

ThemeResolver，主题解析器，负责解析web应用中可用的主题。

MultipartResolver，支持上传等。

FlashMapManager，FlashMap管理器，能够存储并取回两次请求之间的FlashMap对象。可用于在请求之间传递数据。

### 默认的DispatcherServlet配置
DispatcherServlet有一个默认列表，保存在`org.springframework.web.servlet`下
的`DispatcherServlet.properties`文件中。

### DispatcherServlet的处理流程
请求经过DispatcherServlet，此时会按照以下次序对请求进行处理：

* 首先搜索引用的上下文对象WebApplicationContext，并作为一个属性绑定到请求上，以便控制器和其他组件能使用它。属性名默认：`DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE`。
* 将地区locale解析器绑定到请求上。
* 将主题theme解析器绑定到请求上。
* 如果配置了multipart文件处理器，框架会查找该文件是不是multipart的，若是，则将请求包装成一个MultipartHttpServletRequest对象，以便其他组件做进一步处理。
* 为该请求找一个合适的处理器，如果找到处理器，则与该处理器关联的整条执行链都会被执行，包括前置处理器，后置处理器，控制器等。以完成相应模型的准备或视图的渲染。
* 如果处理器返回的是一个model模型，那么框架将渲染相应的视图。若没有返回，则不会渲染视图，此时认为对请求的处理可能已经由处理链完成了。

如果处理请求的过程中抛出了异常，上下文WebApplicationContext对象中定义的异常处理器会负责捕获异常。可以定义自己的异常处理器。

可以定制DispatcherServlet的配置，在web.xml中，添加servlet初始化参数，init-param，可选参数：

contextClass，任意实现了WebApplicationContext接口的类。会初始化该servlet所需要用到的上下文对象。默认框架会用XmlWebApplicationContext对象。

contextConfigLocation，上下文配置文件路径，该值传递给contextClass所指定的上下文实例对象。可包含多个，以逗号分隔。

namespace，WebApplicationContext的命名空间，默认是`[servlet-name]-servlet`

## 控制器Controller的实现
控制器作为应用程序逻辑的处理入口，它会负责去调用已经实现的一些服务。控制器会接收并解析用户的请求，然后转换成一个模型交给视图，由视图渲染出页面最终呈献给用户。

### 使用@Controller注解定义一个控制器
`@controller`注解表明了一个类是作为控制器的角色而存在的。

DispatcherServlet会扫描所有注解了`@Controller`的类，检测其中通过`@RequestMapping`注解配置的方法。

需要在配置文件中添加代码来开启对注解控制器的自动检测：

```
<context:component-scan	base-package="org.springframework.samples.petclinic.web"/>
```

### 使用@RequestMapping注解映射请求路径
使用`@RequestMapping`注解请求URL，映射到类上或某个特定的处理器方法上。

#### 可消费的媒体类型
可以指定一组可消费的媒体类型，缩小映射范围，只有当请求头中Content-Type的值与指定的可消费的媒体类型中有相同的时候才会被匹配。

`consumes="application/json"`

可消费媒体类型的表达式还可以使用否定，MediaType类中定义了一些常量可使用。

#### 可生产的媒体类型
只有当请求头中Accept的值与指定的可生产媒体类型中有相同时，请求才能被匹配。

使用produces条件还可以确保用于生成响应response的内容与指定的可生产的媒体类型是相同的。

也可以使用豆丁表达式。

#### 请求参数与请求头的值
可以通过筛选请求参数的条件来缩小请求匹配范围：

```
@RequestMapping(path="/xxx/xxx",method=RequestMethod.GET,params="myParam=myValue")
```

还可以筛选请求头：

```
headers="myHeader=myValue"
```

### 定义@RequestMapping注解的处理方法（handler method）
#### 支持的方法参数类型