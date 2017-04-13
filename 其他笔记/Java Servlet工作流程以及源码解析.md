关于Servlet的学习还是在上学的时候，自学Java，也就是只是了解了Servlet是什么以及怎么使用，现在慢慢的明白很多很多的框架等等都是在Servlet上做的扩展，也开始明白自己的基础不好。现在回头来学习一下Servlet的相关知识。

# Servlet的使用
对于Servlet怎么写，以及在Servlet容器（这里特指Tomcat）中怎么配置几乎都忘记了，先回头了下一个最简单的Servlet的开发。

这里以一个最简单的自定义Servlet显示一个页面来作为例子。

首先写一个TestServlet，继承HttpServlet：

```
package me.cxis.servlet;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.PrintWriter;

/**
 * Created by cheng.xi on 2017-04-12 23:42.
 */
public class TestServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("doGet...");
        resp.setContentType( "text/html;charset=UTF-8" );
        PrintWriter out = resp.getWriter();
        try {
            out.println( "<html>" );
            out.println( "<head>" );
            out.println( "<title>TestServlet</title>" );
            out.println( "</head>" );
            out.println( "<body>" );
            out.println( "<h2>testServlet</h2>" );
            out.println( "</body>" );
            out.println( "</html>" );
        } finally {
            out.close();
        }
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("doPost...");
        this.doGet(req,resp);
    }

}
```

然后在web.xml中配置刚才写的servlet：

```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    <servlet>
        <servlet-name>testServlet</servlet-name>
        <servlet-class>me.cxis.servlet.TestServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>testServlet</servlet-name>
        <url-pattern>/testServlet</url-pattern>
    </servlet-mapping>
</web-app>
```
这就可以了，启动Servlet容器，这里是tomcat，然后访问`http://localhost:8080/testServlet`就可以看到结果了。

写到这里感觉大学时光又回来了，那时候做项目的时候，可没少写了servlet，老师，学长，实验室，三年，几个人每天就写代码，太单纯！

# Servlet的工作流程
这里只看下一个请求到来时候Servlet是怎么处理的流程，不涉及容器的启动初始化之类的。
## 大概流程
1. 客户端发起一个http请求，比如get类型。
2. Servlet容器接收到请求，根据请求信息调用相应的Servlet。
3. Servlet来处理具体的业务逻辑，也就是我们写的Servlet中的代码。
4. Servlet处理完成之后，返回给Servlet容器。
5. Servlet容器将最后结果返回给客户端。

## 具体流程

1. 客户端发起一个http请求，比如get类型。
2. Servlet容器接收到请求，根据请求信息，封装成HttpServletRequest和HttpServletResponse对象。
3. Servlet容器调用HttpServlet的init()方法，init方法只在第一次请求的时候被调用。
4. Servlet容器调用service()方法。
5. service()方法根据请求类型，这里是get类型，分别调用doGet或者doPost方法，这里调用doGet方法。
6. doXXX方法中是我们自己写的业务逻辑。
7. 业务逻辑处理完成之后，返回给Servlet容器，然后容器将结果返回给客户端。
8. 容器关闭时候，会调用destory方法

这其中的2,3,4,5,6使我们要关注的，其他的步骤是容器实现的，先不了解具体信息。

注意：

- 同一个Servlet只会被初始化一次，也就是init方法只会被调用一次。
- 同一个Servlet只存在一个实例，而可能会有多个请求同时请求一个Servlet，所以存在线程安全问题。

# Servlet生命周期
Servlet生命周期由容器来管理，大致包含了四个阶段：

1. 加载和实例化，由容器负责加载和实例化Servlet。
2. 初始化，容器会调用init方法初始化Servlet对象，init方法只会被调用一次。
3. 处理请求，容器会调用service方法进行请求的处理，service会调用相应的doXxx方法来处理。
4. 服务销毁，即一个Servlet实例从服务中被移除的时候，会调用destory方法，这个方法也只会被执行一次。

下面我们解析的是从初始化开始，对于加载和实例化不做说明。

# Servlet流程的源码分析
## 初始化，init
请求到来时候，会由容器先处理，然后调用Servlet的init方法进行初始化，init方法在GenericServlet中：

```
@Override
public void init(ServletConfig config) throws ServletException {
	//config是由容器处理的，里面包含了请求和Servlet的相关配置信息
    this.config = config;
    //由子类进行实现的init方法
    //我们可以重写init方法，进行一些资源的初始化等的操作
    this.init();
}
```

有关init方法只会执行一次的问题，这里并没有体现到，这是具体的实现，而有关判断是在容器的StandardWrapper类中，会判断是否已经实例化，没有实例化就调用实例化方法实例Servlet，这就会调用init方法。

## 处理请求service
当初始化完成之后，容器会进行处理，然后容器再去调用Servlet的service方法去进行请求的处理，首先进入HttpServlet的service方法：

```
protected void service(HttpServletRequest req, HttpServletResponse resp)
        throws ServletException, IOException {
		//请求类型方法
        String method = req.getMethod();
		//对每种类型分别进行处理
        if (method.equals(METHOD_GET)) {//get方法
        	//get方法涉及到缓存的问题，会对最后修改时间进行判断
            long lastModified = getLastModified(req);
            //不支持lastModified，直接调用doGet方法进行处理
            if (lastModified == -1) {
                // servlet doesn't support if-modified-since, no reason
                // to go through further expensive logic
                doGet(req, resp);
            } else {
            	//支持lastModified
                long ifModifiedSince;
                try {
                	//从请求头中获取If-Modified-Since属性的值
                    ifModifiedSince = req.getDateHeader(HEADER_IFMODSINCE);
                } catch (IllegalArgumentException iae) {
                    // Invalid date header - proceed as if none was set
                    ifModifiedSince = -1;
                }
                //比较时间
                if (ifModifiedSince < (lastModified / 1000 * 1000)) {
                    //Servlet的修改时间晚，表示数据比客户端新
                    //需要修改时间，并调用get方法处理
                    maybeSetLastModified(resp, lastModified);
                    doGet(req, resp);
                } else {
                	//没有修改 直接返回状态给客户端
                    resp.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
                }
            }

        } else if (method.equals(METHOD_HEAD)) {//Head方法
            long lastModified = getLastModified(req);
            maybeSetLastModified(resp, lastModified);
            doHead(req, resp);

        } else if (method.equals(METHOD_POST)) {//post方法
            doPost(req, resp);

        } else if (method.equals(METHOD_PUT)) {//put方法
            doPut(req, resp);

        } else if (method.equals(METHOD_DELETE)) {//delete方法
            doDelete(req, resp);

        } else if (method.equals(METHOD_OPTIONS)) {//options方法
            doOptions(req,resp);

        } else if (method.equals(METHOD_TRACE)) {//trace方法
            doTrace(req,resp);

        } else {
            //servlet不支持其他类型方法
            String errMsg = lStrings.getString("http.method_not_implemented");
            Object[] errArgs = new Object[1];
            errArgs[0] = method;
            errMsg = MessageFormat.format(errMsg, errArgs);

            resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED, errMsg);
        }
    }
```

可以看到，service方法处理请求，会根据请求的类型分别进行处理，get会涉及到最后修改时间问题。这些doXxx方法都会有默认的实现，如果子类不做重写就会执行默认方法。

请求处理完成之后会回到容器中由容器进行其他的处理。

## 销毁方法destory
如果我们重写了destory方法，在容器关闭或者Servlet实例移除的时候，会回调我们的destory方法，一般用来释放资源，这个方法也只会被调用一次。

上面只是最简单的一个Servlet流程，没有涉及到更多的内容，比如上下文，比如session，filter等等，还有一个就是线程安全问题，这个问题会在下面文章继续说明，流程的解析就到这里。