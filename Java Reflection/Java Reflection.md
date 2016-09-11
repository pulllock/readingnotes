[地址](http://ifeve.com/java-reflection-tutorial/)

# Java反射指南
Java反射机制可以让我们在编译期(Compile Time)之外的运行期(Runtime)检查类，接口，变量以及方法的信息。反射还可以让我们在运行期实例化对象，调用方法，通过调用get/set方法获取变量的值。

例子：

```
Method[] methods = MyObject.class.getMethods();
for(Method method : methods){
    System.out.println("Class MyObject method = " + method);
}
```

# Classes
使用Java反射机制可以在运行时期检查Java类的信息，可以获取以下信息：

- Class对象
- 类名
- 修饰符
- 包信息
- 父类
- 实现的接口
- 构造器
- 方法
- 变量
- 注解

## Class对象
检查一个类的信息之前，先获取类的Class对象。Java中所有类型包括基本类型，数据都有关联的Class类对象。

获取类的Class对象：

```
Class clazz = MyObject.class;
```
如果编译期不知道类的名字，但是可以在运行期得到类名字符串，可以通过如下来获取Class对象：

```
String className = "";
Class clazz = Class.forName(className);
```

使用Class.forName()方法时，必须提供一个类的全名。会抛ClassNotFoundException异常。

## 类名

通过getName()方法返回类的全限定名：

```
Class clazz = ...//获取Class对象
String className = clazz.getName();
```

getSimpleName()发挥类名，不包含包名：

```
Class clazz = ...//获取Class对象
String className = clazz.getSimpleName();
```

## 修饰符
可通过Class对象来访问一个类的修饰符：

```
Class clazz = ...
int modifiers = clazz.getModifiers();
```
修饰符被包装成一个int类型的数字，每个修饰符都是一个位标识(flag bit)，这个位标识可以设置和清除修饰符的类型。

可以使用java.lang.reflect.Modifier类中的方法来检查修饰符的类型：

```
Modifier.isAbstract(int modifiers);
Modifier.isFinal(int modifiers);
Modifier.isInterface(int modifiers);
Modifier.isNative(int modifiers);
Modifier.isPrivate(int modifiers);
Modifier.isProtected(int modifiers);
Modifier.isPublic(int modifiers);
Modifier.isStatic(int modifiers);
Modifier.isStrict(int modifiers);
Modifier.isSynchronized(int modifiers);
Modifier.isTransient(int modifiers);
Modifier.isVolatile(int modifiers);
```

## 包信息
使用Class对象可获取包信息：

```
Class clazz = ...
Package package = clazz.getPackage();
```

## 父类

```
Class superclass = aClass.getSuperclass();
```
## 实现的接口

获取实现的接口类集合：

```
Class clazz = ...
Class[] interfaces = clazz.getInterfaces();
```
getInterfaces()方法仅仅只返回当前类所实现的接口。当前类的父类如果实现了接口，这些接口是不会在返回的Class集合中的，尽管实际上当前类其实已经实现了父类接口。

## 构造器

```
Constructor[] constructors = aClass.getConstructors();
```
## 方法

```
Method[] method = aClass.getMethods();
```

## 变量

```
Field[] field = clazz.getFields();
```
## 注解

```
Annotation[] annotations = clazz.getAnnotations();
```

# 构造器
利用反射可以检查一个类的构造方法，可以在运行期创建一个对象。

## 获取Constructor

```
Class clazz = ...
Constructor[] constructors = clazz.getConstructors();
```NoSuchMethodException
返回的Constructor数组包含每一个声明为公有的（Public）构造方法。

返回的构造方法的方法参数为String类型：

```
Class clazz = ...
Construstor constructor = clazz.getContructor(new Class[]{String.class});
```

没有满足的会抛异常NoSuchMethodException。

## 构造方法参数

获取参数信息：

```
Constructor constructor = ...
Class[] parameterTypes = constructor.getParameterTypes();
```

## 实例化类

```
Constructor constructor = MyObject.class.getConstructor(String.class);
MyObject myObject = (MyObject)constructor.newInstance("arg1");
```

constructor.newInstance()方法的方法参数是一个可变参数列表，但是当你调用构造方法的时候你必须提供精确的参数，即形参与实参必须一一对应。

# 变量
可以运行期检查一个类的变量信息(成员变量)或者获取或者设置变量的值。

## 获取Field对象

```
Class clazz = ...
Field[] fields = clazz.getFields();
```
返回的Field对象数组包含了指定类中声明为公有的(public)的所有变量集合。

获取指定的变量：

```
Class clazz = ...;
Field field = clazz.getField("someField");//someField为变量名
```

没有找到匹配的 抛异常NoSuchFieldException。

## 变量名称

```
Field field = ...
String fieldName = field.getName();
``
## 变量类型

```
Field field = clazz.getField("someField");
Object fieldType = field.getType();
```
# 获取设置变量值
通过调用Field.get()或Field.set()方法，获取或者设置变量的值：

```
Class clazz = MyObject.class;
Field field = clazz.getField("someField");
MyObject object = new MyObject();
Object value = field.get(object);

field.set(object,value);
```

传入Field.get()/Field.set()方法的参数objet应该是拥有指定变量的类的实例。

如果变量是静态变量的话(public static)那么在调用Field.get()/Field.set()方法的时候传入null做为参数而不用传递拥有该变量的类的实例。

# 方法

## 获取Method对象

```
Class clazz = ...;
Method[] methods = clazz.getMethods();
```
返回的Method对象数组包含了指定类中声明为公有的(public)的所有变量集合。


通过参数类型来获取指定的方法：

```
Class clazz = ...;
Method method = clazz.getMethod("methodName",new Class[]{String.class});
```

无法找到匹配的，抛异常NoSuchMethodException。


获取的方法没有参数，那么在调用getMethod()方法时第二个参数传入null即可

```
Class clazz = ...;
Method method = clazz.getMethod("methodName",null);
```
## 方法参数以及返回类型
获取指定方法参数：

```
Method method = ...;
Class[] parameterTypes = method.getParameterTypes();
```
获取返回类型：

```
Method method = ...;
Class returnType = method.getReturnType();
```

## 通过Method对象调用方法

```
Method method = MyObject.class.getMethod("methodName",String.class);
Object returnValue = method.invoke(null,"value1");
```
传入的null参数是你要调用方法的对象，如果是一个静态方法调用的话则可以用null代替指定对象作为invoke()的参数。

# Getters and Setters
需要检查一个类所有的方法来判断哪个方法是getters和setters。

# 私有变量和私有方法
Java反射可以从对象的外部访问私有变量以及方法。

## 访问私有变量
可以调用Class.getDeclaredField(String name)方法或者Class.getDeclaredFields()方法。

Class.getField(String name)和Class.getFields()只会返回公有的变量，无法获取私有变量。

```
PrivateObject object = new PrivateObject("Private Object");
Field field = ProvateObject.class.getDeclaredField("privateString");
field.setAccessible(true);
String fieldValue = (String) field.get(object);
```

## 访问私有方法
需要调用 Class.getDeclaredMethod(String name, Class[] parameterTypes)或者Class.getDeclaredMethods() 方法。

Class.getMethod(String name, Class[] parameterTypes)和Class.getMethods()方法，只会返回公有的方法，无法获取私有方法。

```
PrivateObject object = new PrivateObject("private object");
Method method = PrivateObject.class.getDeclaredMethod("methodName",null);
method.setAccessible(true);
String returnValue = method.invoke(object,null);
```
# 注解
定义注解：

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)

public @interface MyAnnotation {
  public String name();
  public String value();
}
```
@Retention(RetentionPolicy.RUNTIME)表示这个注解可以在运行期通过反射访问。如果你没有在注解定义的时候使用这个指示那么这个注解的信息不会保留到运行期，这样反射就无法获取它的信息。

@Target(ElementType.TYPE) 表示这个注解只能用在类型上面（比如类跟接口）。你同样可以把Type改为Field或者Method，或者你可以不用这个指示，这样的话你的注解在类，方法和变量上就都可以使用了。

# 类注解

```
Class clazz = TestClass.class;
Annotation[] annotations = clazz.getAnnotations();
for(Annotation annotation : annotations){
	if(annotation instanceof MyAnnotation){
		MyAnnotation myAnnotation = (Myannotation) annotation;
	}
}
```

## 方法注解

```
Method method = ...
Annotation annotations = method.getDeclaredAnnotations();
```

## 参数注解
通过Method对象来访问方法参数注解：

```
Method method = ...
Annotation[] [] paranmeterAnnotations = method.getParameterAnnotations();
Class[] parameterTypes = method.getParameterTypes();
int i=0;
for(Annotation[] annotations : parameterAnnotations){
  Class parameterType = parameterTypes[i++];

  for(Annotation annotation : annotations){
    if(annotation instanceof MyAnnotation){
        MyAnnotation myAnnotation = (MyAnnotation) annotation;
        System.out.println("param: " + parameterType.getName());
        System.out.println("name : " + myAnnotation.name());
        System.out.println("value: " + myAnnotation.value());
    }
  }
}
```

需要注意的是Method.getParameterAnnotations()方法返回一个注解类型的二维数组，每一个方法的参数包含一个注解数组。

## 变量注解

```
Field field = ... //获取方法对象
Annotation[] annotations = field.getDeclaredAnnotations();

for(Annotation annotation : annotations){
 if(annotation instanceof MyAnnotation){
 MyAnnotation myAnnotation = (MyAnnotation) annotation;
 System.out.println("name: " + myAnnotation.name());
 System.out.println("value: " + myAnnotation.value());
 }
}
```

# 泛型
## 泛型方法返回类型

如果你获得了java.lang.reflect.Method对象，那么你就可以获取到这个方法的泛型返回类型信息。如果方法是在一个被参数化类型之中（译者注：如T fun()）那么你无法获取他的具体类型，但是如果方法返回一个泛型类（译者注：如List fun()）那么你就可以获得这个泛型类的具体参数化类型。

```
public class MyClass {

  protected List<String> stringList = ...;

  public List<String> getStringList(){
    return this.stringList;
  }
}

Method method = MyClass.class.getMethod("getStringList", null);

Type returnType = method.getGenericReturnType();

if(returnType instanceof ParameterizedType){
    ParameterizedType type = (ParameterizedType) returnType;
    Type[] typeArguments = type.getActualTypeArguments();
    for(Type typeArgument : typeArguments){
        Class typeArgClass = (Class) typeArgument;
        System.out.println("typeArgClass = " + typeArgClass);
    }
}

```
这段代码会打印出 “typeArgClass = java.lang.String”，Type[]数组typeArguments只有一个结果 – 一个代表java.lang.String的Class类的实例。Class类实现了Type接口。

## 泛型方法参数类型

```
public class MyClass {
  protected List<String> stringList = ...;

  public void setStringList(List<String> list){
    this.stringList = list;
  }
}

method = Myclass.class.getMethod("setStringList", List.class);

Type[] genericParameterTypes = method.getGenericParameterTypes();

for(Type genericParameterType : genericParameterTypes){
    if(genericParameterType instanceof ParameterizedType){
        ParameterizedType aType = (ParameterizedType) genericParameterType;
        Type[] parameterArgTypes = aType.getActualTypeArguments();
        for(Type parameterArgType : parameterArgTypes){
            Class parameterArgClass = (Class) parameterArgType;
            System.out.println("parameterArgClass = " + parameterArgClass);
        }
    }
}
```

## 泛型变量类型
可以通过反射来访问公有（Public）变量的泛型类型，无论这个变量是一个类的静态成员变量或是实例成员变量。

```
public class MyClass {
  public List<String> stringList = ...;
}

Field field = MyClass.class.getField("stringList");

Type genericFieldType = field.getGenericType();

if(genericFieldType instanceof ParameterizedType){
    ParameterizedType aType = (ParameterizedType) genericFieldType;
    Type[] fieldArgTypes = aType.getActualTypeArguments();
    for(Type fieldArgType : fieldArgTypes){
        Class fieldArgClass = (Class) fieldArgType;
        System.out.println("fieldArgClass = " + fieldArgClass);
    }
}
```

# 数组
Java反射机制通过java.lang.reflect.Array这个类来处理数组。

## 创建一个数组

```
int array = (int[]) Array.newInstance(int.class,3);
```
3 表示数组空间是多大。

## 访问数组
可以使用Array.get(…)和Array.set(…)方法来访问。

```
int[] array = (int[])Array.newInstance(int.class,3);

Array.set(array,0,123);
...

Array.get(array,0);
```

## 获取数组的Class对象
不通过反射获取数组的Class对象：

```
Class stringArrayClass = String[].class;
```
获取int数组的Class对象：

```
Class intArray = Class.forName("[I");
```
获取String类型：

```
Class stringArrayClass = Class.forName("[Ljava.lang.String;");
```
不能通过Class.forName()方法获取一个原生数据类型的Class对象。

通常会用下面这个方法来获取普通对象以及原生对象的Class对象：

```
public Class getClass(String className){
  if("int" .equals(className)) return int .class;
  if("long".equals(className)) return long.class;
  ...
  return Class.forName(className);
}
```
一旦你获取了类型的Class对象，你就有办法轻松的获取到它的数组的Class对象，你可以通过指定的类型创建一个空的数组，然后通过这个空的数组来获取数组的Class对象。这样做有点讨巧，不过很有效。如下例：

```
Class theClass = getClass(theClassName);
Class stringArrayClass = Array.newInstance(theClass, 0).getClass();
```

## 获取数组的成员类型
可以通过Class.getComponentType()方法获取这个数组的成员类型。

```
String[] strings = new String[3];
Class stringArrayClass = strings.getClass();
Class stringArrayComponentType = stringArrayClass.getComponentType();
System.out.println(stringArrayComponentType);
```

## 动态代理
利用Java反射机制你可以在运行期动态的创建接口的实现。java.lang.reflect.Proxy类就可以实现这一功能。

动态的代理的用途十分广泛，比如数据库连接和事物管理（transaction management）还有单元测试时用到的动态mock对象以及AOP中的方法拦截功能等等都使用到了动态代理。

## 创建代理
可以通过Proxy.newProxyInstance()方法创建动态代理，三个参数：

1. 类加载器（ClassLoader）用来加载动态代理类。
2. 一个要实现的接口的数组。
3. 一个InvocationHandler把所有方法的调用都转到代理上。

```
InvocationHandler handler = new MyInvocationHandler();
MyInterface proxy = (MyInterface)Proxy.newProxyInstance(MyInterface.class.getClassLoader(),new Class[]{MyInterface.class},handler);
```

在执行完这段代码之后，变量proxy包含一个MyInterface接口的的动态实现。所有对proxy的调用都被转向到实现了InvocationHandler接口的handler上。

## InvocationHandler接口
所有对动态代理对象的方法调用都会被转向到InvocationHandler接口的实现上，下面是InvocationHandler接口的定义：

```
public interface InvocationHandler{
  Object invoke(Object proxy, Method method, Object[] args)
         throws Throwable;
}
```

实现类：

```
public class MyInvocationHandler implements InvocationHandler{
	public Object invoke(Object proxy, Method method, Object[] args)
	  throws Throwable {
	    //do something "dynamic"
	  }
}
```
传入invoke()方法中的proxy参数是实现要代理接口的动态代理对象。通常你是不需要他的。

invoke()方法中的Method对象参数代表了被动态代理的接口中要调用的方法，从这个method对象中你可以获取到这个方法名字，方法的参数，参数类型等等信息。

Object数组参数包含了被动态代理的方法需要的方法参数。注意：原生数据类型（如int，long等等）方法参数传入等价的包装对象（如Integer， Long等等）。

## 常见用例
动态代理常被应用到一下几种情况：

- 数据库连接以及事务管理
- 单元测试中的动态Mock对象
- 自定义工厂与依赖注入容器之间的适配器
- 类似AOP的方法拦截

### 数据库连接以及事务管理
方法调用序列如下：

```
web controller --> proxy.execute(...);
  proxy --> connection.setAutoCommit(false);
  proxy --> realAction.execute();
    realAction does database work
  proxy --> connection.commit();
```
# 动态类加载与重载
Java允许你在运行期动态加载和重载类。

## 类加载器
所有Java应用中的类都是被java.lang.ClassLoader类的一系列子类加载的。因此要想动态加载类的话也必须使用java.lang.ClassLoader的子类。

一个类一旦被加载时，这个类引用的所有类也同时会被加载。类加载过程是一个递归的模式，所有相关的类都会被加载。但并不一定是一个应用里面所有类都会被加载，与这个被加载类的引用链无关的类是不会被加载的，直到有引用关系的时候它们才会被加载。

## 类加载体系
当你新创建一个标准的Java类加载器时你必须提供它的父加载器。当一个类加载器被调用来加载一个类的时候，首先会调用这个加载器的父加载器来加载。如果父加载器无法找到这个类，这时候这个加载器才会尝试去加载这个类。

## 类加载
顺序如下：

1. 检查这个类是否已经被加载。
2. 如果没有被加载，则首先调用父加载器加载。
3. 如果父加载器不能加载这个类，则尝试加载这个类。

## 动态类加载
获取一个类加载器然后调用它的loadClass()方法：

```
public class MainClass{
	public static void main(String args[]){
		ClassLoader classLoader = MainClass.class.getClassLoader();
		try {
			Class aClass = classLoader.loadClass("com.jenkov.MyClass");
			System.out.println("aClass.getName() = " + aClass.getName());
		    } catch (ClassNotFoundException e) {
			e.printStackTrace();
		    }
	}
}
```

## 动态类重载
Java内置的类加载器在加载一个类之前会检查它是否已经被加载。因此重载一个类是无法使用Java内置的类加载器的，如果想要重载一个类你需要手动继承ClassLoader。

在你定制ClassLoader的子类之后，你还有一些事需要做。所有被加载的类都需要被链接。这个过程是通过ClassLoader.resolve()方法来完成的。由于这是一个final方法，因此这个方法在ClassLoader的子类中是无法被重写的。resolve()方法是不会允许给定的ClassLoader实例链接一个类两次。所以每当你想要重载一个类的时候你都需要使用一个新的ClassLoader的子类。你在设计类重载功能的时候这是必要的条件。

## 自定义类重载
不能使用已经加载过类的类加载器来重载一个类。因此你需要其他的ClassLoader实例来重载这个类。

所有被加载到Java应用中的类都以类的全名（包名 + 类名）作为一个唯一标识来让ClassLoader实例来加载。这意味着，类MyObject被类加载器A加载，如果类加载器B又加载了MyObject类，那么两个加载器加载出来的类是不同的。

```
MyObject object = (MyObject)
    myClassReloadingFactory.newInstance("com.jenkov.MyObject");
```
如果myClassReloadingFactory工厂对象使用不同的类加载器重载MyObject类，你不能把重载的MyObject类的实例转换（cast）到类型为MyObject的对象变量。一旦MyObject类分别被两个类加载器加载，那么它就会被认为是两个不同的类，尽管它们的类的全名是完全一样的。你如果尝试把这两个类的实例进行转换就会报ClassCastException。

你可以解决这个限制，不过你需要从以下两个方面修改你的代码：

1. 标记这个变量类型为一个接口，然后只重载这个接口的实现类。
2. 标记这个变量类型为一个超类，然后只重载这个超类的子类。
