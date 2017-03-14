设计模式中的代理模式，使用代理对象来完成真实对象对用户请求的处理，用户请求每次都是访问到代理对象，而访问不到真实对象，只有代理对象才能访问到真实对象。

使用代理模式，可以由代理类对外部提供统一的接口，代理类可以在对真实类操作的时候进行附加操作，进行扩展等。可以在远程调用中需要使用代理类处理远程方法调用的细节。

代理模式中的角色：

- Subject，这是一个接口，定义了代理类和真实类的公共方法，需要他们实现此接口。
- RealSubject，这是Subject的实现类，真正做业务逻辑的地方。
- Proxy，代理类。

动态代理是指在运行时动态生成代理类。生成动态代理类有很多方式：Java动态代理，CGLIB，Javassist，ASM库等。

Java动态代理中，每一个动态代理类都必须要实现InvocationHandler接口，当我们通过代理对象调用一个方法的时候，这个方法就会被转发给代理类的invoke方法来调用，InvocationHandler接口只有一个方法：

```
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
```
proxy，真实对象。method，我们调用的真实对象的方法。args，要调用真实对象的方法时接受的参数。

实现了InvocationHandler的类是动态代理类，而Proxy就是用来动态创建代理类对象的工具，通常情况下我们只使用newProxyInstance这个方法，newProxyInstance定义：

```
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) throws IllegalArgumentException {}
```
loader，对生成的代理类对象进行加载的ClassLoader。interfaces，代理对象会实现这些接口。h，动态代理对象调用方法的时候，要发送到哪个InvocationHandler上。

延迟加载代理举例：

接口类：

```
public interface ILoadFile{
	String load();
}
```

真实实现类：

```
public class LoadFile implements ILoadFile{
	String fileContent = "";
    
    public LoadFile(){
    	try{
        	//这里模拟加载一个大文件，需要时间很长
        	Thread.sleep(10000);
            //加载完之后
            fileContent = "file contents...";
        }catch(Exception e){
        
        }
    }
    
    public String load(){
    	return fileContent;
    }
    
}
```

代理类：

```
public LoadFileHandler implements InvocationHandler{
	ILoadFile loadFile = null;
    
    public Object invoke(Object proxy,Method method,Object[] args) throws Throwable{
    	if( loadFile == null){
        	loadFile = new LoadFile();
        }
        
        return loadFile.load();
    }
}
```

生成动态代理对象并使用：

```
public static void main(String[] args){
	ILoadFile loadFile = (ILoadFile)Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(),new Class[]{ILoadFile.class},new LoadFileHandler());
    loadFile.load();
}
```

