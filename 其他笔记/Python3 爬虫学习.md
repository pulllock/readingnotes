# Python3 爬虫学习

## urllib

`urllib.request` 用来发送request和获取request的结果。
`urllib.error` 针对异常的模块
`urllib.parse`解析处理URL
`urllib.robotparse`解析robot.txt文件

### urllib.request

`urllib.request.urlopen()` 提供了最基本的构造HTTP请求的方法，可以模拟请求的发起过程。还能处理authentication授权验证，redirections重定向，cookies，以及其他内容。

对于HTTP 和 HTTPS返回的是http.client.HTTPResponse对象。
