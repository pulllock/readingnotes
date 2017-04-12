动态代理是指在运行时动态生成代理类。不需要我们像静态代理那个去手动写一个个的代理类。生成动态代理类有很多方式：Java动态代理，CGLIB，Javassist，ASM库等。这里主要说一下Java动态代理的实现。

# Java动态代理
## InvocationHandler接口
Java动态代理中，每一个动态代理类都必须要实现InvocationHandler接口，当我们通过代理对象调用一个方法的时候，这个方法就会被转发给代理类的invoke方法来调用，InvocationHandler接口只有一个方法：

```
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
```
proxy，真实对象。method，我们调用的真实对象的方法。args，要调用真实对象的方法时接受的参数。

## Proxy类
实现了InvocationHandler的类是动态代理类，而Proxy就是用来动态创建代理类对象的工具，通常情况下我们只使用newProxyInstance这个方法，newProxyInstance定义：

```
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) throws IllegalArgumentException {}
```
loader，对生成的代理类对象进行加载的ClassLoader。interfaces，代理对象会实现这些接口。h，动态代理对象调用方法的时候，要发送到哪个InvocationHandler上。

# 例子
我们知道，在之前的静态代理中，每个代理类都实现了特定接口，针对每一个事情都需要去定义一个代理类，会迅速的使类变多，重复也会多。而使用动态代理可以避免这一点，还是使用之前的找人还钱的例子。

Subject：

```
package me.cxis.test.gof.proxy.dynamicproxy;

/**
 * Created by cheng.xi on 2017-04-12 19:58.
 * 代理模式的主题类，这里代表的是欠钱的人
 */
public interface Subject {
    /**
     * 还给我的钱
     * @param moneyCount 欠钱数
     * @return 还给我的钱数
     */
    int giveMeMyMoney(int moneyCount);
}
```

RealSubject：

```
package me.cxis.test.gof.proxy.dynamicproxy;

/**
 * Created by cheng.xi on 2017-04-12 19:58.
 * 真实主题，这里代表的是欠我钱的人
 */
public class RealSubject implements Subject {
    @Override
    public int giveMeMyMoney(int moneyCount) {
        return moneyCount;
    }
}
```

DynamicProxy：

```
package me.cxis.test.gof.proxy.dynamicproxy;


import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

/**
 * Created by cheng.xi on 2017-04-12 19:59.
 * 代理
 */
public class DynamicProxy implements InvocationHandler{

    private Object target;

    public DynamicProxy(Object target){
        this.target = target;
    }

    public <T> T getProxy(){
        return (T) Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("办事之前先收取点费用");
        System.out.println("开始办事");
        Object result = method.invoke(target,args);
        System.out.println("办完了");
        return result;
    }
}
```

测试方法：

```
package me.cxis.test.gof.proxy.dynamicproxy;

/**
 * Created by cheng.xi on 2017-04-12 20:06.
 */
public class Main {

    public static void main(String[] args) {
        Subject zhangsan = new RealSubject();
        Subject proxy = new DynamicProxy(zhangsan).getProxy();
        int money = proxy.giveMeMyMoney(1000);
        System.out.println(money);
    }
}
```
可以看到我们把之前的代理类，换成了现在的动态代理类，调用方法也有所改变，看起来没什么大的区别，但是当我们在需要一个类似的业务的时候，就有差别了，我们无需在定义第二个动态代理类，只需要有新的Subject1接口和RealSubject1实现即可，在我们测试方法中直接调用就可以了，动态代理类完全不需要改变。

## Java动态代理的优缺点
优点：Java动态代理可以避免静态代理带来的代码冗余的问题。

缺点：Java动态代理只能针对接口创建代理，不能针对类创建代理。

# Java动态代理到底做了什么？
其实动态代理是在运行时候为我们生成了一个代理类，大概如下：

```
public final class $Proxy1 extends Proxy implements Subject{
    private InvocationHandler h;
    private $Proxy1(){}
    public $Proxy1(InvocationHandler h){
        this.h = h;
    }
    public int giveMeMyMoney(int i){
    	////创建method对象
        Method method = Subject.class.getMethod("giveMeMyMoney", new Class[]{int.class});
        //调用了invoke方法
        return (Integer)h.invoke(this, method, new Object[]{new Integer(i)}); 
    }
}
```
从中可以看到，InvocationHandler是我们实现的那个类，我们实现了invoke方法，这里就是调用了invoke方法。

同时我们看到这个类实际上实现了我们的主题类Subject，看着和静态代理差不多了把。另外着重看下这个类继承了Proxy类，看到这里，对于JDK动态代理为什么不能代理类只能代理接口，就明了了，因为他已经继承了Proxy类。

# 动态代理的用处
- Spring AOP就是使用的动态代理方式。
- dubbo消费者初始化的时候生成代理，也是使用的动态代理。
- hibernate的懒加载。

# 其他例子

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

