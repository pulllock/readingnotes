服务提供者初始化完成之后，对外暴露的是Exporter。服务消费者初始化完成之后，对外提供的是Proxy代理。

服务消费者发起请求：

1. helloService.hello()，消费者需要调用的远程的服务，这里helloService是代理。
2. helloServiceSub.hello()，proxy代理被Stub包装。
3. helloServiceProxy.hello()，消费者端Invoker代理。
4. InvokerInvocationHandler.invoke()，由代理类执行的invocationHandler。
5. MockClusterInvoker.invoke()，集群对集群Invoker进行的包装，Mock功能。
6. ClusterInvoker.invoke(),执行集群伪装的Invoker。
7. Directory.list()，查找伪装在集群后面的所有Invoker。
8. Router.route()，通过路由策略从list中选择一些Invoker。
9. LoadBalance.select()，通过负载均衡策略中选择一个Invoker执行。
10. Filter.invoke()，执行所有消费者端Filter。
11. Invoker.invoke()，执行消费者端的Invoker，即DubboInvoker。
12. HeaderExchangerClient.request()，信息交换Client执行请求，其中封装了NettyClient，HeaderExchangerChannel。
13. HeaderExchangerChannel.request()，此类封装了NettyClient，信息交换通道创建Request执行请求，并创建DefaultFuture返回。
14. NettyClient的父类AbstractClient.send()，消费者Client执行请求。
15. NettyChannel.send()，Netty通道执行请求发送。
16. Channel.write(),框架通知执行写操作，触发Handler。
17. NettyHandler.writeRequested()，执行NIO框架的顶级Handler。
18. NettyCodecAdapter.encode()执行NIO框架的编码逻辑。
19. NettyClient的父类AbstractPeer.send()，执行装饰了Handler的NettyClient。
20. MultiMessageHandler.send()，多消息处理Handler
21. HeartbeatHandler.send()，心跳处理Handler
22. AllChannelHandler,send()，消息派发器Handler
23. DecodeHandler，编码Handler
24. HeaderExchangerHandler,send()，信息交换Handler
25. requestHandler父类ChannelHandlerAdapter.send()，最后执行的Handler

以上从18开始的执行的Handler在消费者端是没有意义的，因为从Channel.write()开始，提供者就已经接收到消息进行处理了。

服务提供者处理并响应请求的过程：

1. NettyCodecAdapter.messageReceived()通过NIO框架通信，接收到消费者发送的消息。
2. NettyHandler.messageReceived()提供者端的NIO顶级Handler处理
3. NettyServer.received()NIO框架的Server接受请求信息
4. MultiMessageHandler.received()多消息处理Handler
5. HeartbeatHandler.received()心跳处理
6. AllChannelHandler.received()消息派发器
7. DecodeHandler.received()编码Handler
8. HeaderExchangerHandler.received()信息交换Handler，请求响应模式
9. requestHandler.reply()执行与Exporter交接的最初的Handler
10.  getInvoker()得到提供者端的Exporter，再得到相应的提供者端Invoker
11.  Filter.invoke()执行所有提供者端的Filter，所有附加逻辑均由此完成。
12.  AbstractInvoker.invoke()执行封装了服务实现类的原始Invoker
13.  helloService.hello()执行服务实现类
14.  NettyChannel.send() HeaderExchangerHandler得到执行结果Response再返回给消费者，此代码由HeaderExchangerHandler发起 
15.  Channel.write () NIO框架通知执行写操作，并触发Handler  
16.  NettyHanlder.writeRequested() 执行NIO框架的顶级Handler 
17.  NettyCodecAdapter.encode() 执行NIO框架的编码逻辑 
18.  NettyServer父类AbstractPeer.send() 执行装饰了Handler的NettyServer  
19.  MultiMessageHandler.send() 多消息处理Handler 
20.  HeartbeatHandler.send() 心跳处理Handler 
21.  AllChannelHandler.send() 消息派发器Handler 
22.  DecodeHandler.send() 编码Handler 
23.  HeaderExchangerHandler.send() 信息交换Handler 
24.  requestHandler父类ChannelHandlerAdapter.send() 最后执行Handler

以上Handler的执行在提供者端也是是没有意义，因为Channel.write ()执行时，消费者端已经收到响应消息了，也正在处理了。

消费者接受到服务端返回的响应后的处理过程：

1. NettyCodecAdapter.messageReceived() 提供者通过NIO框架通信并接受提供者发送过来的响应 
2. NettyHandler.messageReceived() 消费者端的NIO顶级Handler处理 
3. NettyClient.received() NIO框架的Client接受响应信息 
4. MultiMessageHandler.received () 多消息处理Handler 
5. HeartbeatHandler.received ()心跳处理Handler 
6. AllChannelHandler.received () /消息派发器Handler 
7. DecodeHandler.received () 编码Handler 
8. HeaderExchangerHandler.received () 信息交换Handler，请求-响应模式 
9. DefaultFuture.received() 设置response到消费者请求的Future中，以供消费者通过DefaultFuture.get()取得提供者的响应，此为同步转异步重要一步，且请求超时也由DefaultFuture控制。 
10. Filter.invoke() 消费者异步得到响应后，DubboInvoker继续执行，从而Filter继续执行，从而返回结果给消费者。 
11.	InvokerInvocationHandler.invoker() 执行invocationHandler后续代码 
12. helloServiceProxy.hello() 消费者端Invoker代理后续代码 
13.	helloServiceSub.hello() 执行后续代码 
14.	helloService.hello() 执行后续代码 
15.	呈现结果给消费者客户端 

消费者端的DubboInvoker发起请求后，后续的逻辑是异步的或是指定超时时间内阻塞的，直到得到响应结果后，继续执行DubboInvoker中逻辑。

对于异步请求时，消费者得到Future，其余逻辑均是异步的。

主要核心逻辑，对于附加功能，大部分由Filter增强完成。消费者还可以通过设置async、sent、return来调整处理逻辑，async指异步还是同步请求，sent指是否等待请求消息发出即阻塞等待是否成功发出请求、return指是否忽略返回值即但方向通信，一般异步时使用以减少Future对象的创建和管理成本。