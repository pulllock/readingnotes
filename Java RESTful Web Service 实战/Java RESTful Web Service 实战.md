# REST API 设计

## REST 统一接口

### GET方法
幂等性和安全性。

抽象层注解资源,JAX-RS 2.0的HTTP方法注解可以定义在接口和POJO中。

JAX-RS 2.0：@GET

### HEAD方法
head方法和get方法相似，服务器端的返回值不包括http实体，head方法也是安全和幂等的。

JAX-RS 2.0：@HEAD

### OPTIONS方法
options方法和get方法类似，安全和幂等。options用于读取资源所支持的所有http请求方法。

JAX-RS 2.0：@OPTIONS

### PUT方法

put方法是一种写操作的http请求，更新或添加资源。

put方法是幂等的，多次插入或者更新同一份数据，在服务器端对资源状态所发生的改变是相同的。

JAX-RS 2.0：@PUT

请求实体媒体类型使用http头的Content Type定义，响应实体媒体类型使用http头的Accept定义。

服务端，@Consumes(MediaType.APPLICATION_XML)定义了服务端要消费的媒体类型，即客户端请求实体的媒体类型。

@Produces(MediaType.TEXT_PLAIN)定义了服务器生产的媒体类型，即服务器产生的响应实体的媒体类型。

### DELETE方法
幂等的，多次删除同一份数据在服务器端产生的改变是相同的。

JAX-RS 2.0：@DELETE

### POST方法
是一种写操作的HTTP请求，RPC所有写操作均使用post方法，rest只使用post添加资源。

不幂等，也不安全。

JAX-RS 2.0：@POST

## REST资源定位
REST使用URI实现资源定位。

### @QueryParam
定义查询参数。

### @PathParam
@PathParam 定义路径参数。
@Path 定义资源路径。可以使用动态变量的方式{参数名称:正则表达式}

@MatrixParam 定义参数。

### @FormParam
定义表单参数。用以处理请求实体媒体类型为Content-Type：application/x-www-form-urlencode的请求。

@Encoded 标识禁用自动解码。

@DefaultValue 提供默认值。






