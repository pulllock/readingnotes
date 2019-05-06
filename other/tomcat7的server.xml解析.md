这里对tomcat7的server.xml文件进行解释一下，方便在分析启动源码的时候理解Digester做的事情。

```
<?xml version='1.0' encoding='utf-8'?>
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <Listener className="org.apache.catalina.security.SecurityListener" />
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <Listener className="org.apache.catalina.core.JasperListener" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <Service name="Catalina">
    <Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
        maxThreads="150" minSpareThreads="4"/>

    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />

    <Connector executor="tomcatThreadPool"
               port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />

    <Connector port="8443" protocol="org.apache.coyote.http11.Http11Protocol"
               maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
               clientAuth="false" sslProtocol="TLS" />

    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />

    <Engine name="Catalina" defaultHost="localhost">

      <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"/>

      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>

      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">

        <Valve className="org.apache.catalina.authenticator.SingleSignOn" />

        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log." suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
    </Engine>
  </Service>
</Server>
```

# Server
tomcat中Server代表一个tomcat实例，所以只会存在一个Server，而在配置文件中也是作为顶级元素出现，代码如下：

```
<Server port="8005" shutdown="SHUTDOWN">
。。。
</Server>
```

- port，监听shutdown命令的端口，-1表示禁用shutdown命令。
- shutdown，关闭tomcat的指令。

# Listener
监听器，用来监听某些事件的发生。

`<Listener className="org.apache.catalina.startup.VersionLoggerListener" />`

VersionLoggerListener，启动时对tomcat，java，操作系统信息打印日志。

`<Listener className="org.apache.catalina.security.SecurityListener" />`

SecurityListener，启动tomcat时，做一些安全检查。

`<Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />`

AprLifecycleListener，用来监听Apache服务器相关的。

`<Listener className="org.apache.catalina.core.JasperListener" />`

JasperListener，Jasper 2 JSP 引擎，主要负责对更新之后的jsp进行重新编译。

`<Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />`

JreMemoryLeakPreventionListener，防止内存溢出的监听器。

`<Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />`

GlobalResourcesLifecycleListener，初始化定义在元素GlobalNamingResources下的全局JNDI资源

`<Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />`

ThreadLocalLeakPreventionListener，防止ThreadLocal溢出监听器。

# GlobalNamingResources
GlobalNamingResources定义Server的全局JNDI资源。可以为所有的引擎应用程序引用。

```
<GlobalNamingResources>
  <Resource name="UserDatabase" auth="Container"
        type="org.apache.catalina.UserDatabase"
        description="User database that can be updated and saved"
        factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
        pathname="conf/tomcat-users.xml" />
</GlobalNamingResources>
```
配置文件中定义了一个JNDI，名为UserDatabase，通过`conf/tomcat-users.xml`的内容，来得到一个用于授权用户的数据库，是一个内存数据库。

# Service

```
<Service name="Catalina">
。。。
</Service>
```
Server下面可以有多个Service，Service下面有多个Connector和一个Engine。这里默认的Service名字为Catalina，下面有两个Connector：Http和AJP。

- name，Service显示的名称，名字必须唯一。

# Connector

```
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```
上面是用来处理http请求的Connector。

- port，端口号8080。
- protocol，协议，http协议
- connectionTimeout，响应的最大等待时间，20秒
- redirectPort，ssl请求会重定向到8443端口

```
<Connector executor="tomcatThreadPool"
           port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```
上面是使用线程池，处理http请求。


```
<Connector port="8443" protocol="org.apache.coyote.http11.Http11Protocol"
           maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
           clientAuth="false" sslProtocol="TLS" />
```

上面处理ssl请求，端口是8443。


```
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
```
上面处理AJP请求，可以将tomcat和apache的http服务器一起运行。

# Engine
Engine是容器，一个Service中只包含一个Engine：

```
<Engine name="Catalina" defaultHost="localhost">
...
</Engine>
```
Engine下面可以包含一个多或者多个Host。Engine从http请求的头信息中的主机名或者ip映射到真确的主机上。

- name，Engine的名字，需要唯一。
- defaultHost，默认主机名

# Cluster
集群相关的配置。tomcat支持服务器集群，可以复制整个集群的回话和上下文属性，也可以部署一个war包到所有的集群上。

```
<Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"/>
```

# Realm

```
<Realm className="org.apache.catalina.realm.LockOutRealm">
  <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
         resourceName="UserDatabase"/>
</Realm>
```
Realm是一个包含user、password、role的数据库，Realm可以定义在任何容器中。这里通过外部资源UserDatabase进行认证。

# Host

```
<Host name="localhost"  appBase="webapps"
      unpackWARs="true" autoDeploy="true">

  <Valve className="org.apache.catalina.authenticator.SingleSignOn" />

  <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
         prefix="localhost_access_log." suffix=".txt"
         pattern="%h %l %u %t &quot;%r&quot; %s %b" />
</Host>
```
Host虚拟主机，定义在Engine下面，一个Engine下面可以有多个Host，在一个Host下面可以有多个Context。

- name，虚拟主机的网络名称，必须有一个host的名字和Engine的defaulHost一样。
- appBase，虚拟主机应用的根目录，默认是webapps。
- unpackWARs，在webapps目录下的war文件是否应该解压。
- autoDeploy，值为true时，tomcat会定时检查appBase等目录，对新的web应用和Context描述文件进行部署。

# Value

```
<Valve className="org.apache.catalina.authenticator.SingleSignOn" />

<Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
       prefix="localhost_access_log." suffix=".txt"
       pattern="%h %l %u %t &quot;%r&quot; %s %b" />
```
Value在这里是阀门的意思，可以拦截http请求，可以定义在任何容器中。

SingleSignOn 是单点登录，AccessLogValve是访问日志的记录。
