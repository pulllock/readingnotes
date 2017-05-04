主要记录一下http协议的一些简单的知识，主要包括请求消息，响应消息的组成，以及get和post的‘对比’，对于更详细的信息可以看下http RFC。https也没有做说明。

http基于请求响应模式，无状态，应用层的协议，特点如下：

- 支持C/S模式。
- 无连接，每次连接只处理一个请求，服务器处理完请求，并返回给客户端之后，就会断开连接。
- 无状态，指的是协议对于事务处理没有记忆能力，如果需要前面的信息，需要重传。

## http请求消息（Request）
http请求由三部分组成：请求行，请求头，请求体。比如下面的例子：

```
GET / HTTP/1.1
Host: cxis.me
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.81 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.6,en;q=0.4,zh-TW;q=0.2,nb;q=0.2,da;q=0.2
```

`GET / HTTP/1.1`是请求行，GET是请求方法，`/`是请求资源在服务器上的路径，`HTTP/1.1`是http协议版本号。

剩下的是请求头，格式为`xxxx: value`。

这里是get方法，所有没有请求体，请求体是向服务器提交的数据，在请求头和请求体中间会有一行空行。

## http响应消息（Response）
响应消息包括：响应行，响应头，响应体。一个响应消息如下：

```
HTTP/1.1 200 OK
Server: GitHub.com
Content-Type: text/html; charset=utf-8
Last-Modified: Wed, 03 May 2017 08:32:57 GMT
Access-Control-Allow-Origin: *
Expires: Wed, 03 May 2017 13:51:42 GMT
Cache-Control: max-age=600
Content-Encoding: gzip
X-GitHub-Request-Id: AACA:40A3:65A338:8EB391:5909DE15
Content-Length: 9905
Accept-Ranges: bytes
Date: Wed, 03 May 2017 14:41:45 GMT
Via: 1.1 varnish
Age: 0
Connection: keep-alive
X-Served-By: cache-nrt6130-NRT
X-Cache: HIT
X-Cache-Hits: 1
X-Timer: S1493822505.031176,VS0,VE181
Vary: Accept-Encoding
X-Fastly-Request-ID: bc385cef3dbff07f200175fa461c920cb6ca4b3f
```
`HTTP/1.1 200 OK`是响应行，`HTTP/1.1`是http协议版本号，200是状态码，OK是状态消息，和响应码对应。

剩下的是响应头，这里没有响应体。GET方法的响应体为空。

## 请求方法

- GET，获取被Request-URI指定的信息。
- POST，向服务器提交数据。
- HEAD，获取响应消息报头。
- PUT，请求服务器保存一个资源。
- DELETE，请求服务器删除资源。
- TRACE，请求服务器回应收到的请求消息。
- OPTIONS，查询相关的资源和选项。
- CONNECT，预留关键字，现在没有用。

## GET和POST对比
GET和POST我觉得不应该硬拿来对比，他们是http规范定义的两种不同的方法，各有各的用处，为什么要对比呢？

### 关于定义
GET是获取资源，是幂等的；POST是提交资源，是非幂等的。它们是http协议里面定义的两个不同的方法。

### 关于缓存
GET请求的响应是可缓存的，但是需要响应满足HTTP缓存的要求。POST响应是不可缓存的，除非响应里面有Cache-Control或者Expires属性。

### 关于请求数据
GET方法会把请求的数据附加到URL之后，也就是放到请求行中；POST则是把提交的数据放到请求体中。因此在地址栏可以直接看到GET请求提交的参数，而看不到POST请求的参数。

### 关于安全
通常我们说的有关安全，只是相对的安全，比如说GET方法能直接在地址栏看到参数，而POST不能。这通常让人认为是安全和不安全的区别，其实如果抓包或者其他手段一样可以看到GET和POST提交的数据，两者并没有什么安全可言。

### 关于数据长度
http协议并没有对传输的数据大小做限制，也没有对URL长度做限制，所以从http协议本身来说并没有长度的限制。而我们通常说的URL或者数据的长度限制其实是浏览器或者服务器的限制。

对于GET请求来说，提交的数据都会在URL中，各浏览器对URL的限制不太一样，所以没有什么标准可言；对于POST请求来说，数据存放在请求体中，并没有长度限制，但是服务器通常会有对POST提交数据的大小限制，因此也没有标准可言。

### 关于POST两次请求
对于GET请求，浏览器会把请求头和请求体一起发送；而对于POST请求，浏览器会先发送请求头，服务器响应`100 continue`之后，浏览器再发送请求体。

## 状态码
在响应消息的状态行中有一个状态码和状态消息，两者是对应的，状态码总共有五大类：

- 1xx，做指示信息，表示请求被接收到，继续处理。
- 2xx，成功，表示被成功接收，理解，接受。
- 3xx，重定向，为了完成请求必须采取进一步的动作。
- 4xx，客户端错误，请求有语法错误或者请求无法实现。
- 5xx，服务端错误，服务器未能实现请求。

而具体的状态码有很多，不在这里一一列举，下面是一些常用到的：

- 200 OK，表示请求成功
- 400 Bad Request 客户端错误，有语法错误
- 401 Unauthorized，请求未授权
- 403 Forbidden，服务器拒绝服务
- 404 Not Found，资源不存在
- 405 Method Not Allowed，方法不被允许
- 500 Internal Server Error，服务器内部错误
- 502 Bad Gateway，网关错误
- 503 Service Unavailable，服务不可用

## 消息报头
在请求消息的第二部分是请求头，在响应消息的第二部分是响应头，请求头和响应头又叫做消息报头，这是可选的。其实消息报头不只是包括请求头和响应头，一个消息报头包括：普通报头、请求报头、响应报头、实体报头。

下面列出了各种报头，含义没有一一列出，如有需要可以查看[http RFC](https://tools.ietf.org/html/rfc2616)
### 普通报头
普通报头既适用于请求消息也适用于响应消息，这些头域不适用于实体传输，只适用于传输消息。

- Cache-Control 控制缓存指令，缓存指令是单向，独立的。
- Connection 允许发送指定连接的选项
- Date 消息产生的日期和时间
- Pragma
- Trailer
- Transfer-Encoding
- Upgrade
- Via
- Waring

### 请求报头

- Accept 指定客户端接受哪些类型的信息
- Accept-Charset 指定客户端接受的字符集
- Accept-Encoding 指定客户端可接受的内容编码
- Accept-Language 指定客户端可接受的语言
- Authorization 客户端有权限查看某个资源
- Expect
- From
- Host 指定被请求资源的主机和端口号
- If-Match
- If-Modified-Since
- If-None-Match
- If-Range
- If-Unmodified-Since
- Max-Forwards
- Proxy-Authorization
- Range
- Referer
- TE
- User-Agent
### 响应报头

- Accept-Ranges
- Age
- ETag
- Location
- Proxy-Authenticate
- Retry-After
- Server
- Vary
- WWW-Authenticate
### 实体报头

- Allow
- Content-Encoding
- Content-Language
- Content-Length 指明实体正文的长度，以字节方式存储的十进制数字来表示
- Content-Location
- Content-MD5
- Content-Range
- Content-Type 指明发送给接收者的实体正文的媒体类型
- Expires 响应过期的日期和时间
- Last-Modified 用于指示资源的最后修改日期和时间
- extension-header
