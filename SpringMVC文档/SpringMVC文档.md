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

### 使用@RequestParam将请求参数绑定至方法参数

```
public String setupForm(@RequestParam("petId")){

}
```
若参数使用了该注解，则参数默认是必须提供的，但也可以把该参数标注为非必须的，需要将@RequestParam注解的required属性设为false。

### 使用@RequestBody注解映射请求体
请求体到方法参数的转换是由HttpMessageConvert完成的。

### 使用@ResponseBody注解映射响应体
@ResponseBody可被应用与方法上，标志该方法的返回值应该被直接写回到HTTP响应体中去。

### 使用@RestController注解创建REST控制器
结合了@ResponseBody和@Controller。

可与@ControllerAdvice配合使用。

### 使用HTTP实体HttpEntity
HttpEntity和@ResponseBody，@RequestBody相似，除了能获得请求体和响应体中的内容，还可以存取请求头和响应头。

### 对方法使用@ModelAttribute注解
@ModelAttribute可被应用在方法或参数上。注解在方法上的@ModelAttribute说明方法的作用是用于添加一个或多个属性到model上。这样的方法能接收与@RequestMapping注解相同的参数类型，只不过不能被直接映射到具体的请求上。在同一个控制器中，@ModelAttribute的方法实际上会在@RequestMapping方法钱被调用。

常用来填充一些公共需要的属性或数据。

一个控制器可以又数量不限的@ModelAttribute方法，多会在@RequestMapping方法之前被调用。

@ModelAttribute注解也可以用在@RequestMapping方法上，这种情况，方法的返回值将会被解释为model的一个属性，而非一个视图名。

### 在方法参数上使用@ModelAttribute
被注解在参数上，说明了该参数的值将由model中取得，如果model中找不到，会先被实例化，然后被添加到model中。model中存在以后，请求中所有名称匹配的参数多会填充到该参数上。这称为数据绑定。

可以在注解了@ModelAttribute的参数后紧跟着一个BindingResult参数，用于记录数据绑定过程中的错误。

### 在请求间使用@SessionAttributes注解，使用会话保存模型数据
类级别的@SessionAttributes声明了某个特定处理器所使用的会话属性。

### 使用@CookieValue注解映射cookie值
@CookieValue注解能将一个方法参数与一个HTTP cookie的值进行绑定。

### 使用@RequestHeader注解映射请求头属性
@RequestHeader注解能将一个方法参数与一个请求头属性进行绑定。

## 方法参数与类型转换
从请求参数，路径变量，请求头属性或者cookie中抽取出来的String类型的值，可能需要被转换成其所绑定的目标方法参数或字段的类型。所有简单类型int，long，Date等都有内置实现，想定制转换过程，可以通过WebDataBinder或者为Formatters配置一个FormattingConversionService来做到。

### 定制WebDataBinder的初始化
想通过Spring的WebDataBinder在属性编辑器中做请求参数的绑定，可以使用在控制器内使用@InitBinder注解的方法，在注解了@ControllerAdvice的类中使用@InitBinder注解的方法，或者提供一个定制的WebBindingInitializer。

#### 数据绑定的定制：使用@InitBinder
@InitBinder标记的方法，会初始化一个WebDataBinder并用以为处理器方法，填充命令对象和表单对象的参数。

绑定器初始化方法不能有返回值。

### 异步请求的处理
SpringMVC3.2引入了给予Servlet3的异步请求处理，控制器方法不一定需要返回一个值，可以返回一个Java.util.concurrent.Callable对象，并通过SpringMVC所偶管理的线程来产生返回值。

Callable的异步请求被处理时发生的事件：

* 控制器先返回一个Cllable对象。
* SpringMVC开始进行异步处理，把该Callable对象交给另一个独立线程的执行器TaskExecutor处理。
* DispatcherServlet和所有过滤器都退出Servlet容器线程，此时方法的响应对象仍未返回。
* Callable对象最终产生一个返回结果，此时SpringMVC会重新把请求分派回Servlet容器，恢复处理。
* DispatcherServlet再次被调用，恢复对Callable异步处理所返回结果的处理。

#### 异步请求的异常处理
若控制器返回的Callable在执行过程出现异常，会被正常的异常处理流程捕获处理。

#### 拦截异步请求
HandlerInterceptor可以实现AsyncHandlerInterceptor接口拦截异步请求，在异步请求开始时，被调用的回调方法是afterConcurrentHandlingStarted方法，而非一般的postHandle和afterCompletion方法。

#### HTTP Streaming
可以让方法返回一个ResponseBodyEmitter类型的对象来实现HTTP Streaming，该对象可被用于发送多个对象。

通常@ResponseBody只能返回一个对象，是通过HttpMessageConverter写到响应体中的。

#### 使用服务端时间推送的HTTP Streaming
SseEmitter是ResponseBodyEmitter的子类，提供了对服务器端事件推送的技术支持。只需要返回一个SseEmitter对象即可。

#### 直接写回输出流OutputStream的HTTP Streaming
通过返回一个StreamingResponseBody类型的对象来实现。

