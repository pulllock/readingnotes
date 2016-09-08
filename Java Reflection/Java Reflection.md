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
