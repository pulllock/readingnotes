html同源策略是不允许JavaScript的跨域请求的，而使用CORS（Cross-origin resource sharing）可以实现跨域请求，当然也有其他的办法，常用的有JSONP方式来实现跨域，这里就简单的列举一下实现跨域的几种办法，对于CROS和JSONP详细的了解一下。

# CROS
CROS可以实现跨域，是HTML5的标准，需要浏览器的支持，同时也需要服务器端的支持。使用CROS实现跨域，主要的工作是在后端实现，需要在服务器端做设置，一般都是设置http头中的`Access-Control-Allow-Origin`属性，用来指定哪些站点可以访问。

## CROS常用属性

- `Access-Control-Allow-Origin`，允许哪些站点访问。
- `Access-Control-Max-Age`，表示多久之内不需要在发送预检请求，有关预检请求下面说明。
- `Access-Control-Allow-Methods`，表示允许的请求方法，比如get、post、delete等。
- `Access-Control-Allow-Headers`，表示允许的content-type。
- `Access-Control-Allow-Credentials`，表示允许请求发送Cookie。
- `Access-Control-Expose-Headers`，表示允许的Header。

## CROS中简单请求和非简单请求

CROS中简单请求规则：

- 请求方法是HEAD，GET或者POST。
- HTTP头中只能包括以下几种：Accept，Content-Type，Content-Language，Accept-Language，Last-Event-ID

比如有一个get方法，请求头中包括的信息为以上列举的，当浏览器发送请求时发现这是一个简单请求，就会在发送的请求头中自动添加一个Origin表示请求发送的源地址。请求头如下：

```
GET /test HTTP/1.1
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0
Host: www.aaa.com
Origin: http://www.bbb.com
```
服务端接受到请求之后，根据Origin字段判断是否允许该请求，如果服务端允许，则在Http的头信息中添加`Access-Control-Allow-Origin`以及服务端配置的其他属性，然后返回正确的结果；如果服务端不允许该请求，不会在头信息中添加`Access-Control-Allow-Origin`属性。

服务端返回结果后，浏览器接收到请求，根据`Access-Control-Allow-Origin`来决定是否拦截该请求。如果没有这个属性，就会出错，有这个属性就可以正常处理。

非简单请求就是除了上面的简单请求之外的，都是非简单请求。非简单请求的跨域操作，其实会有两次请求到服务端，一次是预检请求（Prelight request），另外一次才是真正的请求。

比如当发送一个DELETE请求时，浏览器会发现这是一个非简单请求，就会先发送一个预检请求，请求如下：

```
OPTIONS /test HTTP/1.1
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0
Host: www.aaa.com
Origin: http://www.bbb.com
Access-Control-Request-Method: DELETE
```

请注意预检请求使用的是OPTIONS，预检请求会询问服务器是否允许Origin的DELETE方法，允许的话就可以回应了。

浏览器接收到预检请求的回应之后，会根据返回的头信息判断，不允许的话就报错，允许的话就可以发送正常请求了，正在的CORS请求和简单请求一样。

# 过滤器的形式使用CROS
通常可以使用过滤器的形式来实现CROS，只需要实现Filter接口，然后把自定义的过滤器配置到web.xml中即可。

```
public class CorsFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        response.setHeader("Access-Control-Allow-Origin", "*");
        response.setHeader("Access-Control-Allow-Methods", "GET,POST,DELETE,PUT,OPTIONS");
        response.setHeader("Access-Control-Allow-Credentials", true);
        response.setHeader("Access-Control-Allow-Headers"," Content-Type,X-Token");
        response.setHeader("Access-Control-Max-Age", "3600");
        response.setHeader("Access-Control-Expose-Headers", "X-My-Header");
        chain.doFilter(req, res);
    }

    @Override
    public void destroy() {
    }
}
```

需要在web.xml中配置过滤器：

```
<!--支持跨域访问-->
<filter>
<filter-name>corsFilter</filter-name>
<filter-class>tb.admin.api.cors.CorsFilter</filter-class>
</filter>
<filter-mapping>
	<filter-name>corsFilter</filter-name>
	<url-pattern>/*</url-pattern>
</filter-mapping>
```

服务器端设置好了CROS相关配置后，其他前端就可以跨域访问了。

# SpringMVC中使用CROS

SpringMVC中使用CROS需要到4.2版本之后，并且使用很方便，如果版本对应的话，可以优先考虑使用Spring的CORS配置。

## 使用@CrossOrigin注解

Spring4.2中提供了`@CrossOrigin`注解，用来实现CROS，`@CrossOrigin(origins = "http://localhost:9000")`直接用在Controller中的方法上，也可以用在类级别上。该注解默认允许所有的origins，所有的headers，所有在`@RequestMapping`中指定的方法，maxAge默认为30分钟。（具体的可以看下Spring相关源码。）

## 使用全局CORS配置

还可以使用全局的CORS配置，继承WebMvcCOnfigurerAdapter来实现：

```
public class CorsConfigurerAdapter extends WebMvcConfigurerAdapter{
	@Override
	public void addCorsMappings(CorsRegistry registry) {
		registry.addMapping("/*").allowedOrigins("http://localhost:9000");
	 }
}
```
其他的配置也可以依次添加，然后将该类注入到容器中即可。

# nginx中配置CORS
如果使用了nginx的话，也可以在nginx中配置CORS来实现跨域请求，在nginx.conf里找到server项,并添加如下配置：

```
location / {
    add_header 'Access-Control-Allow-Origin' '*';
    add_header 'Access-Control-Allow-Credentials' 'true';
    add_header 'Access-Control-Allow-Headers' 'Authorization,Content-Type';
    add_header 'Access-Control-Allow-Methods' 'GET,POST,PUT,DELETE,OPTIONS';
...
}
```
nginx具体的测试没有做，猜测原理应该跟上面类似，暂先不做过多解释。

# JSONP方式实现跨域
在CROS没有出现之前，JSONP方式实现跨域请求十分常见。JSONP原理实际上是对script标签的利用。需要服务端和前端都做处理，现在感觉耦合性有点大了，尤其是跟上面的CORS对比起来。

JSONP只支持get请求，但是它能够支持老的浏览器。

# 其他方法
其他方法还有使用`document.domain`，src标签，navigation对象，以及html5中的`window.postMessage`，这里都不做讲解，条件允许，尽量使用新的CORS来解决跨域问题。
