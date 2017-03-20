（这里做的解析不是很详细，等到走完整个流程再来解析）Dubbo中编解码的工作由Codec2接口的实现来处理，回想一下第一次接触到Codec2相关的内容是在服务端暴露服务的时候，根据具体的协议去暴露服务的步骤中，在DubboProtocol的createServer方法中：

```
private ExchangeServer createServer(URL url) {
	。。。
    //这里url会添加codec=dubbo
    url = url.addParameter(Constants.CODEC_KEY, Version.isCompatibleVersion() ? COMPATIBLE_CODEC_NAME : DubboCodec.NAME);
    ExchangeServer server;
    try {
        server = Exchangers.bind(url, requestHandler);
    }
    。。。
    return server;
}
```
紧接着进入`Exchangers.bind(url, requestHandler);`：

```
public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
	//如果url中没有codec属性，就会添加codec=exchange
    url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
    return getExchanger(url).bind(url, handler);
}
```

然后会继续进入HeaderExchanger的bind方法：

```
public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
    return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
}
```
在这里会创建一个DecodeHandler实例。继续跟踪Transporters的bind方法，会发现直接返回一个NettyServer实例，在NettyServer的父类AbstractEndpoint构造方法初始的时候，会根据url获取一个ChannelCodec，并将其赋值给codec存放到NettyServer的实例中。

我们先看下`getChannelCodec(url);`方法：

```
protected static Codec2 getChannelCodec(URL url) {
	//获取codecName，不存在的话，默认为telnet
    String codecName = url.getParameter(Constants.CODEC_KEY, "telnet");
    //先看下是不是Codec2的实现，是的话就根据SPI扩展机制获得Codec2扩展的实现
    //我们这里默认使用的是DubboCountCodec
    if (ExtensionLoader.getExtensionLoader(Codec2.class).hasExtension(codecName)) {
        return ExtensionLoader.getExtensionLoader(Codec2.class).getExtension(codecName);
    } else {
    	//如果不是Codec2的实现，就去查找Codec的实现
        //然后使用CodecAdapter适配器类来转换成Codec2
        return new CodecAdapter(ExtensionLoader.getExtensionLoader(Codec.class)
                                           .getExtension(codecName));
    }
}
```
这里返回的是Codec2，而Codec这个接口已经被标记为过时。到这里的话，在NettyServer中就会存在一个Codec2的实例了。

在继续往下看到NettyServer中的doOpen()方法，这里是使用Netty的逻辑打开服务并绑定监听服务的地方：

```
protected void doOpen() throws Throwable {
    NettyHelper.setNettyLoggerFactory();
    ExecutorService boss = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerBoss", true));
    ExecutorService worker = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerWorker", true));
    ChannelFactory channelFactory = new NioServerSocketChannelFactory(boss, worker, getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS));
    bootstrap = new ServerBootstrap(channelFactory);

    final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
    channels = nettyHandler.getChannels();
    bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
        public ChannelPipeline getPipeline() {
        	//这里的getCodec方法获取到的codec就是在AbstractEndpoint中我们获取到的codec
            //NettyCodecAdapter，适配器类
            NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec() ,getUrl(), NettyServer.this);
            ChannelPipeline pipeline = Channels.pipeline();
            pipeline.addLast("decoder", adapter.getDecoder());//SimpleChannelUpstreamHandler
            pipeline.addLast("encoder", adapter.getEncoder());//OneToOneEncoder
            pipeline.addLast("handler", nettyHandler);
            return pipeline;
        }
    });
    // bind
    channel = bootstrap.bind(getBindAddress());
}
```
这里就在Netty的pipeline中添加了编解码器。这里涉及到Netty的相关流程，可以先了解下[Netty3服务端流程简介](http://cxis.me/2017/03/20/Netty3%E6%9C%8D%E5%8A%A1%E7%AB%AF%E6%B5%81%E7%A8%8B%E7%AE%80%E4%BB%8B/)。

decoder为解码器，是一个SimpleChannelUpstreamHandler，从Socket到Netty中的时候，需要解码，也就是服务提供端接收到消费者的请求的时候，需要解码。

encoder是编码器，是OneToOneEncoder，这个类实现了ChannelDownstreamHandler，从服务提供端发送给服务消费者的时候，需要编码。

nettyHandler实现了ChannelUpstreamHandler, ChannelDownstreamHandler两个，上下的时候都需要处理。

接收到服务消费者的请求的时候，会先执行decoder，然后执行nettyHandler。

发送给消费者的时候，会先执行nettyHandler，然后执行encoder。

## dubbo协议头

![dubbo协议头示意图](Dubbo中编码和解码的解析/dubbo_protocol_header.jpg)

协议头是16字节的定长数据：

- 2字节short类型的Magic
- 1字节的消息标志位
	
    - 5位序列化id
    - 1位心跳还是正常请求
    - 1位双向还是单向
    - 1位请求还是响应
    
- 1字节的状态位
- 8字节的消息id
- 4字节数据长度

## 编码的过程

首先会判断是请求还是响应，代码在ExchangeCodec的encode方法：

```
public void encode(Channel channel, ChannelBuffer buffer, Object msg) throws IOException {
    if (msg instanceof Request) {//Request类型
        encodeRequest(channel, buffer, (Request) msg);
    } else if (msg instanceof Response) {//Response类型
        encodeResponse(channel, buffer, (Response) msg);
    } else {//telenet类型的
        super.encode(channel, buffer, msg);
    }
}
```
### 服务提供者对响应信息编码
在服务提供者端一般是对响应来做编码，所以这里重点看下encodeResponse。

encodeResponse：

```
protected void encodeResponse(Channel channel, ChannelBuffer buffer, Response res) throws IOException {
    try {
    	//序列化方式
        //也是根据SPI扩展来获取，url中没指定的话默认使用hessian2
        Serialization serialization = getSerialization(channel);
        //长度为16字节的数组，协议头
        byte[] header = new byte[HEADER_LENGTH];
        //魔数0xdabb
        Bytes.short2bytes(MAGIC, header);
        //序列化方式
        header[2] = serialization.getContentTypeId();
        //心跳消息还是正常消息
        if (res.isHeartbeat()) header[2] |= FLAG_EVENT;
        //响应状态
        byte status = res.getStatus();
        header[3] = status;
        //设置请求id
        Bytes.long2bytes(res.getId(), header, 4);
		//buffer为1024字节的ChannelBuffer
        //获取buffer的写入位置
        int savedWriteIndex = buffer.writerIndex();
        //需要再加上协议头的长度之后，才是正确的写入位置
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
        ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
        ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
        // 对响应信息或者错误消息进行编码
        if (status == Response.OK) {
            if (res.isHeartbeat()) {
            	//心跳
                encodeHeartbeatData(channel, out, res.getResult());
            } else {
            	//正常响应
                encodeResponseData(channel, out, res.getResult());
            }
        }
        //错误消息
        else out.writeUTF(res.getErrorMessage());
        out.flushBuffer();
        bos.flush();
        bos.close();
		//写出去的消息的长度
        int len = bos.writtenBytes();
        //查看消息长度是否过长
        checkPayload(channel, len);
        Bytes.int2bytes(len, header, 12);
        //重置写入的位置
        buffer.writerIndex(savedWriteIndex);
        //向buffer中写入消息头
        buffer.writeBytes(header); // write header.
        //buffer写出去的位置从writerIndex开始，加上header长度，加上数据长度
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
    } catch (Throwable t) {
        // 发送失败信息给Consumer，否则Consumer只能等超时了
        if (! res.isEvent() && res.getStatus() != Response.BAD_RESPONSE) {
            try {
                // FIXME 在Codec中打印出错日志？在IoHanndler的caught中统一处理？
                logger.warn("Fail to encode response: " + res + ", send bad_response info instead, cause: " + t.getMessage(), t);

                Response r = new Response(res.getId(), res.getVersion());
                r.setStatus(Response.BAD_RESPONSE);
                r.setErrorMessage("Failed to send response: " + res + ", cause: " + StringUtils.toString(t));
                channel.send(r);

                return;
            } catch (RemotingException e) {
                logger.warn("Failed to send bad_response info back: " + res + ", cause: " + e.getMessage(), e);
            }
        }

        // 重新抛出收到的异常
        if (t instanceof IOException) {
            throw (IOException) t;
        } else if (t instanceof RuntimeException) {
            throw (RuntimeException) t;
        } else if (t instanceof Error) {
            throw (Error) t;
        } else  {
            throw new RuntimeException(t.getMessage(), t);
        }
    }
}
```

### 服务消费者对请求信息编码
消费者端暂先不做解析

## 解码的过程

### 服务提供者对请求消息的解码

decode方法一次只会解析一个完整的dubbo协议包，但是每次收到的协议包不一定是完整的，或者有可能是多个协议包。看下代码解析，首先看NettyCodecAdapter的内部类InternalDecoder的messageReceived方法：

```
public void messageReceived(ChannelHandlerContext ctx, MessageEvent event) throws Exception {
    Object o = event.getMessage();
    if (! (o instanceof ChannelBuffer)) {
        ctx.sendUpstream(event);
        return;
    }

    ChannelBuffer input = (ChannelBuffer) o;
    int readable = input.readableBytes();
    if (readable <= 0) {
        return;
    }

    com.alibaba.dubbo.remoting.buffer.ChannelBuffer message;
    if (buffer.readable()) {
        if (buffer instanceof DynamicChannelBuffer) {
            buffer.writeBytes(input.toByteBuffer());
            message = buffer;
        } else {
            int size = buffer.readableBytes() + input.readableBytes();
            message = com.alibaba.dubbo.remoting.buffer.ChannelBuffers.dynamicBuffer(
                size > bufferSize ? size : bufferSize);
            message.writeBytes(buffer, buffer.readableBytes());
            message.writeBytes(input.toByteBuffer());
        }
    } else {
        message = com.alibaba.dubbo.remoting.buffer.ChannelBuffers.wrappedBuffer(
            input.toByteBuffer());
    }

    NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
    Object msg;
    //读索引
    int saveReaderIndex;
    try {
        do {
            saveReaderIndex = message.readerIndex();
            try {
            //解码
                msg = codec.decode(channel, message);
            } catch (IOException e) {
                buffer = com.alibaba.dubbo.remoting.buffer.ChannelBuffers.EMPTY_BUFFER;
                throw e;
            }
            //不完整的协议包
            if (msg == Codec2.DecodeResult.NEED_MORE_INPUT) {
            	//重置读索引
                message.readerIndex(saveReaderIndex);
                //跳出循环，之后在finally中把message赋值给buffer保存起来，等到下次接收到数据包的时候会追加到buffer的后面
                break;
            } else {//有多个协议包，触发messageReceived事件
                if (saveReaderIndex == message.readerIndex()) {
                    buffer = com.alibaba.dubbo.remoting.buffer.ChannelBuffers.EMPTY_BUFFER;
                    throw new IOException("Decode without read data.");
                }
                if (msg != null) {
                    Channels.fireMessageReceived(ctx, msg, event.getRemoteAddress());
                }
            }
        } while (message.readable());
    } finally {
        if (message.readable()) {
            message.discardReadBytes();
            buffer = message;
        } else {
            buffer = com.alibaba.dubbo.remoting.buffer.ChannelBuffers.EMPTY_BUFFER;
        }
        NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
    }
}
```

继续看`codec.decode(channel, message);`这里是DubboCountCodec的decode方法：

```
public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {
	//当前的读索引记录下来
    int save = buffer.readerIndex();
    //多消息
    MultiMessage result = MultiMessage.create();
    do {
    	//解码消息
        Object obj = codec.decode(channel, buffer);
        //不是完整的协议包
        if (Codec2.DecodeResult.NEED_MORE_INPUT == obj) {
            buffer.readerIndex(save);
            break;
        } else {//多个协议包
            result.addMessage(obj);
            logMessageLength(obj, buffer.readerIndex() - save);
            save = buffer.readerIndex();
        }
    } while (true);
    if (result.isEmpty()) {
        return Codec2.DecodeResult.NEED_MORE_INPUT;
    }
    if (result.size() == 1) {
        return result.get(0);
    }
    return result;
}
```

继续看ExchangeCodec的decode方法：

```
public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {
	//可读字节数
    int readable = buffer.readableBytes();
    byte[] header = new byte[Math.min(readable, HEADER_LENGTH)];
    //协议头
    buffer.readBytes(header);
    //解码
    return decode(channel, buffer, readable, header);
}
```

解码decode：

```
protected Object decode(Channel channel, ChannelBuffer buffer, int readable, byte[] header) throws IOException {
    //检查魔数.
    if (readable > 0 && header[0] != MAGIC_HIGH 
            || readable > 1 && header[1] != MAGIC_LOW) {
        int length = header.length;
        if (header.length < readable) {
            header = Bytes.copyOf(header, readable);
            buffer.readBytes(header, length, readable - length);
        }
        for (int i = 1; i < header.length - 1; i ++) {
            if (header[i] == MAGIC_HIGH && header[i + 1] == MAGIC_LOW) {
                buffer.readerIndex(buffer.readerIndex() - header.length + i);
                header = Bytes.copyOf(header, i);
                break;
            }
        }
        //telenet
        return super.decode(channel, buffer, readable, header);
    }
    //不完整的包
    if (readable < HEADER_LENGTH) {
        return DecodeResult.NEED_MORE_INPUT;
    }

    //数据长度
    int len = Bytes.bytes2int(header, 12);
    checkPayload(channel, len);

    int tt = len + HEADER_LENGTH;
    if( readable < tt ) {
        return DecodeResult.NEED_MORE_INPUT;
    }

    // limit input stream.
    ChannelBufferInputStream is = new ChannelBufferInputStream(buffer, len);

    try {
    	//解码数据
        return decodeBody(channel, is, header);
    } finally {
        if (is.available() > 0) {
            try {
                StreamUtils.skipUnusedStream(is);
            } catch (IOException e) { }
        }
    }
}
```

decodeBody解析数据部分：

```
protected Object decodeBody(Channel channel, InputStream is, byte[] header) throws IOException {
    byte flag = header[2], proto = (byte) (flag & SERIALIZATION_MASK);
    //获取序列化方式
    Serialization s = CodecSupport.getSerialization(channel.getUrl(), proto);
    //反序列化
    ObjectInput in = s.deserialize(channel.getUrl(), is);
    //获取请求id
    long id = Bytes.bytes2long(header, 4);
    //这里是解码响应数据
    if ((flag & FLAG_REQUEST) == 0) {
        //response的id设为来时候的Request的id，这样才能对上暗号
        Response res = new Response(id);
        //判断是什么类型请求
        if ((flag & FLAG_EVENT) != 0) {
            res.setEvent(Response.HEARTBEAT_EVENT);
        }
        //获取状态
        byte status = header[3];
        res.setStatus(status);
        if (status == Response.OK) {
            try {
                Object data;
                if (res.isHeartbeat()) {
                	//解码心跳数据
                    data = decodeHeartbeatData(channel, in);
                } else if (res.isEvent()) {
                	//事件
                    data = decodeEventData(channel, in);
                } else {
                	//响应
                    data = decodeResponseData(channel, in, getRequestData(id));
                }
                res.setResult(data);
            } catch (Throwable t) {
                res.setStatus(Response.CLIENT_ERROR);
                res.setErrorMessage(StringUtils.toString(t));
            }
        } else {
            res.setErrorMessage(in.readUTF());
        }
        return res;
    } else {//这是解码请求数据
        // request的id
        Request req = new Request(id);
        req.setVersion("2.0.0");
        req.setTwoWay((flag & FLAG_TWOWAY) != 0);
        if ((flag & FLAG_EVENT) != 0) {
            req.setEvent(Request.HEARTBEAT_EVENT);
        }
        try {
            Object data;
            if (req.isHeartbeat()) {
            	//心跳
                data = decodeHeartbeatData(channel, in);
            } else if (req.isEvent()) {
            	//事件
                data = decodeEventData(channel, in);
            } else {
            	//请求
                data = decodeRequestData(channel, in);
            }
            req.setData(data);
        } catch (Throwable t) {
            // bad request
            req.setBroken(true);
            req.setData(t);
        }
        return req;
    }
}
```
具体的解码细节交给底层解码器，这里是使用的hessian2。

### 服务消费者对响应消息的解码
暂先不做解释。