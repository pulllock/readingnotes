责任链模式（Chain of Responsibility Pattern），为了避免请求发送者和多个请求处理者耦合在一起，为请求创建一个请求处理者的链，多个请求处理者都在这个链中，并且每个请求处理者都记着下一个请求处理者是谁，请求发生的时候，可以沿着这个链一直向下运行。

责任链模式的角色：

- 抽象处理者（Handler），请求处理者的抽象，定义了处理请求的方法、 持有下一个请求处理者、设置和获取下一个请求处理者的方法。
- 具体处理者（ConcreteHandler），具体的请求处理者实现，可以处理请求或将请求传给下一个请求处理者。

简单代码：

```java
public abstract class Handler {
    protected Handler nextHandler;
    
    public abstract void process();
    
    public Handler getNextHandler() {
        return nextHandler;
    }
    
    public void setNextHandler(Handler nextHandler) {
        this.nextHandler = nextHandler;
    }
}
```

```java
public class ConcreteHandler extends Handler{
    
    @Override
    public void process() {
        if (true) {
            // 当前处理者能处理
            return;
        } 
        
        // 当前处理者不能处理，交给下一个处理者
        if (getNextHandler() == null) {
            return;
        }
        
        getNextHandler().process();
    }
}
```

```java
public class Client {
    public static void main(String[] args) {
        Handler handler1 = new ConcreteHandler1();
        Handler handler2 = new ConcreteHandler2();
        Handler handler3 = new ConcreteHandler3();
        
        handler1.setNextHandler(handler2);
        handler2.setNextHandler(handler3);
        
        handler1.process();
    }
}
```

按照责任链模式的定义，一个请求只能被其中一个请求处理者处理，或者一个请求处理者都无法处理，但是实际应用中大多都是一个请求可以被多个请求处理者处理。

责任链模式的应用：Tomcat中的Filter、Netty中的Pipeline和ChannelHandler、SpringAOP中的Advisor执行、Dubbo的过滤器链、MyBatis中的插件机制、Zuul中的ZuulFilter。

# Spring中使用责任链模式

```java
public abstract class GatewayHandler {

    protected GatewayHandler nextHandler;

    public abstract String process(String request);

    public GatewayHandler getNextHandler() {
        return nextHandler;
    }

    public void setNextHandler(GatewayHandler nextHandler) {
        this.nextHandler = nextHandler;
    }
}
```

```java
@Order(1)
@Component
public class FlowControlHandler extends GatewayHandler {

    @Override
    public String process(String request) {
        GatewayHandler next = getNextHandler();
        if (next == null) {
            return null;
        }
        return next.process(request);
    }
}
```

```java
@Order(2)
@Component
public class BlacklistHandler extends GatewayHandler {

    @Override
    public String process(String request) {
        if (request.contains("hack")) {
            return "ip locked";
        }

        GatewayHandler next = getNextHandler();
        if (next == null) {
            return null;
        }
        return next.process(request);
    }
}
```

```java
@Order(3)
@Component
public class ParamCheckHandler extends GatewayHandler {

    @Override
    public String process(String request) {
        if (request == null || request.contains("error")) {
            return "param error!";
        }

        GatewayHandler next = getNextHandler();
        if (next == null) {
            return null;
        }
        return next.process(request);
    }
}
```

```java
@Order(4)
@Component
public class InvokeServiceHandler extends GatewayHandler {

    @Override
    public String process(String request) {
        // invoke service
        String result = "result for request: " + request;

        GatewayHandler next = getNextHandler();
        if (next == null) {
            return result;
        }
        return next.process(request);
    }
}
```

```java
@Configuration
public class ChainConfig {

    @Resource
    private List<GatewayHandler> gatewayHandlers;

    @PostConstruct
    private void initChain() {
        Collections.sort(gatewayHandlers, AnnotationAwareOrderComparator.INSTANCE);

        int size = gatewayHandlers.size();
        for (int i = 0; i < size; i++) {
            if (i == size - 1) {
                gatewayHandlers.get(i).setNextHandler(null);
            } else {
                gatewayHandlers.get(i).setNextHandler(gatewayHandlers.get(i + 1));
            }
        }
    }
}
```

```java
@RestController
@RequestMapping("/chain")
public class ChainController {

    @Resource
    private List<GatewayHandler> gatewayHandlers;

    @RequestMapping(value = "/test", method = RequestMethod.GET)
    public String test(@RequestParam String str) {
        GatewayHandler defaultHandler = gatewayHandlers.get(0);
        return defaultHandler.process(str);
    }
}
```

# Apache Commons Chain使用

