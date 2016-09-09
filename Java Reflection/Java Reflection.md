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



