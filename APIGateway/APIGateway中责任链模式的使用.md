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

```java
public class CommonContext extends ContextBase {
}
```

```java
public abstract class AbstractCommand implements Command {

    @Override
    public boolean execute(Context context) throws Exception {
        CommonContext commonContext = (CommonContext) context;
        return execute(commonContext);
    }

    public abstract boolean execute(CommonContext context);
}
```

```java
public class BlacklistHandler extends AbstractCommand {

    @Override
    public boolean execute(CommonContext context) {
        String request = String.valueOf(context.get("request"));
        if (request.contains("hack")) {
            context.put("error", "ip locked");
            return true;
        }
        return false;
    }
}
```

```java
public class FlowControlHandler extends AbstractCommand {

    @Override
    public boolean execute(CommonContext context) {
        return false;
    }
}
```

```java
public class ParamCheckHandler extends AbstractCommand {

    @Override
    public boolean execute(CommonContext context) {
        String request = String.valueOf(context.get("request"));
        if (request.contains("error")) {
            context.put("error", "param error!");
            return true;
        }
        return false;
    }
}
```

```java
public class InvokeServiceHandler extends AbstractCommand {

    @Override
    public boolean execute(CommonContext context) {
        String request = String.valueOf(context.get("request"));

        String result = "result for request: " + request;
        context.put("result", result);
        return false;
    }
}
```

```java
public class Chains extends ChainBase {

    public Chains() {
        addCommand(new BlacklistHandler());
        addCommand(new FlowControlHandler());
        addCommand(new ParamCheckHandler());
        addCommand(new InvokeServiceHandler());
    }
}
```

```java
public class Client {

    public static void main(String[] args) throws Exception {
        Chains chains = new Chains();
        CommonContext commonContext = new CommonContext();
        commonContext.put("request", "this hack");
        System.out.println(chains.execute(commonContext));
        System.out.println(JSON.toJSONString(commonContext));

        commonContext = new CommonContext();
        commonContext.put("request", "this error");
        System.out.println(chains.execute(commonContext));
        System.out.println(JSON.toJSONString(commonContext));

        commonContext = new CommonContext();
        commonContext.put("request", "this is normal request");
        System.out.println(chains.execute(commonContext));
        System.out.println(JSON.toJSONString(commonContext));
    }
}
```

# Spring和Commons Chain

```java
public class CommonContext extends ContextBase {
}
```

```java
public abstract class AbstractCommand implements Command {

    @Override
    public boolean execute(Context context) throws Exception {
        CommonContext commonContext = (CommonContext) context;
        return execute(commonContext);
    }

    public abstract boolean execute(CommonContext context);
}
```

```java
@Component
public class BlacklistCommand extends AbstractCommand {

    @Override
    public boolean execute(CommonContext context) {
        System.out.println("BlacklistCommand...");
        String request = String.valueOf(context.get("request"));
        if (request.contains("hack")) {
            context.put("error", "ip locked");
            return true;
        }
        return false;
    }
}
```

```java
@Component
public class FlowControlCommand extends AbstractCommand {

    @Override
    public boolean execute(CommonContext context) {
        System.out.println("FlowControlCommand...");
        return false;
    }
}
```

```java
@Component
public class ParamCheckCommand extends AbstractCommand {

    @Override
    public boolean execute(CommonContext context) {
        System.out.println("ParamCheckCommand...");
        String request = String.valueOf(context.get("request"));
        if (request.contains("error")) {
            context.put("error", "param error!");
            return true;
        }
        return false;
    }
}
```

```java
@Component
public class InvokeServiceCommand extends AbstractCommand {

    @Override
    public boolean execute(CommonContext context) {
        System.out.println("InvokeServiceCommand...");
        String request = String.valueOf(context.get("request"));

        String result = "result for request: " + request;
        context.put("result", result);
        return false;
    }
}
```

```java
public class Chains extends ChainBase {

    public Chains() {
        addCommand(new BlacklistCommand());
        addCommand(new FlowControlCommand());
        addCommand(new ParamCheckCommand());
        addCommand(new InvokeServiceCommand());
    }
}
```

```java
@Configuration
public class CommonsChainConfig {

    @Bean
    public Chains chains() {
        return new Chains();
    }
}
```

```java
@RestController
@RequestMapping("/chain")
public class ChainController {

    @Resource
    private Chains chains;

    @RequestMapping(value = "/commonsChainTest", method = RequestMethod.GET)
    public String commonsChainTest(@RequestParam String str) throws Exception {
        CommonContext commonContext = new CommonContext();
        commonContext.put("request", str);
        System.out.println(chains.execute(commonContext));
        System.out.println(JSON.toJSONString(commonContext));

        String result = commonContext.get("result") == null ? (String) commonContext.get("error") : (String) commonContext.get("result");
        return result;
    }
}
```

