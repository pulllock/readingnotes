在学习Dubbo的时候需要学习Netty的流程等，在此做一个简单的入门学习。Dubbo中使用的是Netty3，所以这里说的都是Netty3。

Netty3可以看成是对Reactor的实现，所以先简单看下Reactor模式。

# Reactor模式
Reactor模式是基于事件驱动的，有以下几种角色存在：

- Handle，句柄，用来表示打开的文件，打开的连接等，Java NIO中使用Channel来表示。
- Synchronous Event Demultiplexer，阻塞的等待发生在句柄上的一个或多个事件，就是监听事件的到来。Java NIO中使用Selector来表示。
- EventHandler接口，来处理不同的请求事件。
- Concrete Event Handler，EventHandler实现。
- Initiation Dispatcher（Reactor），用来管理EventHandler；有事件到来时分发事件到EventHandler上去处理。

# Netty中的Reactor模式
Netty中使用了两层Reactor，Main Reactor用于处理连接请求，Sub Reactor用于处理请求连接之后的读写请求。

# Netty中各类释义

## Channel
Reactor模式中使用Handle来表示打开的连接，也就是事件源，在java nio中使用Channel来抽象事件源，Netty中的Channel是自己的抽象。

## ChannelEvent
在Netty中使用ChannelEvent来抽象在事件源中可以产生的各种事件。

## ChannelHandler
作用就是Reactor模式中的EventHandler，用来处理事件请求。有两个子接口：

- ChannelDownstreamHandler，处理从用户应用流向Netty内部，然后流向Socket的事件。
- ChannelUpstreamHandler，处理从Socket进入Netty内部，然后流向用户应用的事件。

## ChannelPipeline
每个Channel都会有一个ChannelPipeline，用来管理ChannelHandler。ChannelPipeline内部有一个ChannelHandler的双向链表，以Upstream为正方向，Downstream为负方向。

## NioSelector
对应的是Reactor模式中的Synchronous Event Demultiplexer，Java NIO使用Selector，每个Channel都会把自己注册到Selector上，Selector就可以监听Channel中发生的事件。当有事件发生的时候，会生成ChannelEvent实例，该事件会被发送到Channel对应的ChannelPipeline中，然后交给ChannelHandler处理。

NioSelector有两个实现：

- Boss，是Main Reactor，用来处理新连接加入的事件。
- Worker，是Sub Reactor，用来处理各个连接的读写事件。

## ChannelSink
ChannelSink可以看成Handler最后的一个处于末尾的万能handler，只有DownStream包含ChannelSink。

# 服务端例子

```
public class NettyServerTest {

    private final int port;

    public NettyServerTest(int port){
        this.port = port;
    }

    public void startServer(){
        ChannelFactory channelFactory = new NioServerSocketChannelFactory(Executors.newCachedThreadPool(),Executors.newCachedThreadPool());
        ServerBootstrap serverBootstrap = new ServerBootstrap(channelFactory);

        serverBootstrap.setPipelineFactory(new ChannelPipelineFactory() {
            @Override
            public ChannelPipeline getPipeline() throws Exception {
                return Channels.pipeline(new ServerHandlerTest());
            }
        });

        serverBootstrap.bind(new InetSocketAddress(port));
    }

    public static void main(String[] args) {
        new NettyServerTest(8888).startServer();
    }
}
```

```
public class ServerHandlerTest extends SimpleChannelUpstreamHandler {
    @Override
    public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
        ChannelBuffer channelBuffer = (ChannelBuffer)e.getMessage();
        String msg = channelBuffer.toString(Charset.defaultCharset());
        if(msg != null && !"".equals(msg)){
            System.out.println("服务端接收到消息：" + msg);
            ChannelBuffer sendMsg = ChannelBuffers.dynamicBuffer();
            sendMsg.writeBytes("我是服务器，已经接到消息".getBytes());
            e.getChannel().write(sendMsg);
        }else {
            e.getChannel().write("我是服务器，收到了空消息");
        }
        e.getChannel().close();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, ExceptionEvent e) throws Exception {
        e.getCause();
        e.getChannel().close();
    }
}

```

ChannelFactory主要是用来产生Channel实例和ChannelSink实例。

ChannelPipelineFactory主要是用于具体传输数据的处理，是我们自己实现具体内容，一般我们是往里面添加Handler实现。

大概的流程是：

- 首先使用Boss和Worker两个线程池来初始化一个ChannelFactory。
- 使用ChannelFactory来初始化一个ServerBootstrap实例。
- 为ServerBootstrap设置pipelineFactory，这里用来添加各种处理用的Handler。
- 使用Bind方法绑定并监听。

有关具体的分析和源码分析，等到dubbo分析完成之后，再做。