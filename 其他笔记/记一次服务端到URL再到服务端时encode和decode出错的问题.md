# 场景

1. 服务端1先对数据进行rsa加密，然后再进行base64编码
2. 上一步进行编码后的数据附加到url中的一个参数中，进行发送，并在浏览器中进行一次跳转
3. 服务端2接收到上面的url进行base64解码，然后rsa解密

# 出现的错误

服务端2接收到url里面的加密数据之后，进行base64解密，然后rsa解密，出错！

# 原因判断
## 第一次判断
服务端1发送数据之后，浏览器对base64之后的数据进行了encode，服务器2接收到数据没有进行decode，直接base64解码然后rsa解密，出错！

判断是对的，服务器2进行decode之后，再次进行下面的操作，但是发现服务器decode之后数据跟encode之前的不一样，原因分析可能是浏览器对url进行了deceode，并且每个浏览器decode的结果也不一样。

## 第二次判断
服务端1在进行base64之后，再进一步进行对数据进行url encode，这样发送数据之后，浏览器中进行跳转时就不会自动进行encode了，然后服务端2接收到数据之后进行decode，然后出错了！

这次判断也是对的，但是服务器2接收到url数据之后，发现是已经被decode过的数据，原因分析，可能是tomcat自动进行了decode。

# 涉及

这里涉及到大概三个方面的东西要去解决：

1. Java中url encode和decode的了解。
2. 浏览器自动对url进行encode的了解。
3. tomcat对url进行decode的了解。

# 参考
http://www.cnblogs.com/liuhongfeng/p/5006341.html
http://www.jinglingshu.org/?p=3345
https://www.jianshu.com/p/6722d6fe1403
http://blog.csdn.net/vickyway/article/details/46375971
