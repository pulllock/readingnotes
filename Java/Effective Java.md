# 第2章 创建和销毁对象

## 第1条：用静态工厂方法代替构造器

使用静态工厂方法的优势：

1. 有名称。
2. 不必每次调用都创建新对象。
	- 类似享元模式。
	- 实例受控的类：能控制哪些实例应该存在。
3. 可以返回原返回类型的任何子类型的对象。
4. 返回的对象的类可以变化（这和上面一点其实是一样的）
5. 静态工厂方法返回的对象（具体的实现类）可以不用定义，照样可以直接写这个静态工厂方法。这是因为该静态工厂方法的返回值是接口。

第三点：“可以返回原返回类型的任何子类型的对象”，比如：

```java
public static Animal create() {
    return new Dog();
}
```

返回类型是 `Animal`， 实际返回对象是 `Dog`。 `Dog`是 `Animal`的子类型。

“在 Java 8 之前，接口不能有静态方法”，这里涉及到 `Java`接口设计思想的变化。

在 `Java 8`之前，也就是 `Java 7`以及更早的版本中，接口有严格的限制，接口中不能出现静态方法。在 `Java 8`以及之后，接口中可以有静态方法，比如下面的代码：

```java
public interface Animal {

    static Animal create() {
        return new Dog();
    }
}
```

可以直接调用 `Animal`接口中的静态方法：

```java
Animal animal = Animal.create();
```

而在 `Java 8`之前，我们要实现相同功能，一般会放在实现类或者工具类中，比如：

```java
public class Dog {

    public static Animal create() {
        return new Dog();
    }
}
```

这种方式是从 `Java 1.0`就支持的。在 `JDK`中常见的大量的工具类就是这样的功能：

```java
Collections
Arrays
Executors
Paths
Files
```

也可以看作是在弥补“接口不能定义静态方法”的缺陷。

之所以出现这样的变化，是接口的设计思想发生了变化。在 `Java`早期接口定位是：接口只是一个规范。接口只负责告诉别人你必须实现什么功能，而不负责功能怎么实现，因此接口中不应该有实现代码，而静态方法显然属于实现代码，因此接口中也必然不能有静态方法。

随着 `Java`的发展，接口越来越像一种“能力抽象”，有些实现逻辑其实最适合放在接口里的，尤其是一些工厂方法，比如原来的： `Path path = Paths.get("test.txt");`，其实更理想的写法是： `Path path = Path.of("test.txt");`，后面这种写法在 `Java 11`中已经实现了。

所以从 `Java 8`开始接口有了重大的变化：

- 支持静态方法
- 支持默认方法

接口从纯规范变成了规范+部分实现。

另外 `Java 9`中开始允许接口有私有的静态方法，但是静态域和静态成员类仍然需要是公有的。

静态方法代码示例：

```java
public interface Animal {

    static void test() {
        System.out.println("hello");
    }
}
```

调用静态方法：

```java
Animal.test();
```

默认方法代码示例：

```java
public interface Animal {

    default void sleep() {
        System.out.println("sleep");
    }
}
```

接口的实现类无需重写这个默认方法。

第五点：“静态工厂的第五大优势在于，方法返回的对象所属的类，在编写包含该静态工厂方法的类时可以不存在。”，这句话其实是说，静态工厂方法的返回值是一个接口，具体要返回的实现类可以不存在。比如说系统中只存在一个接口：

```java
public interface Payment {
    void pay();
}
```

我仍旧可以定义一个静态工厂方法：

```java
public class PaymentFactory {

    public static Payment create() {
        ...
    }

}
```

但是这个 `Payment`接口的具体实现类可以不存在在这个系统中。

上述 `PaymentFactory`可以像如下的：

```java
public class PaymentFactory {

    public static Payment create() {
        return loadProvider();
    }

    private static Payment loadProvider() {
        ...
    }
}
```

客户端使用：

```java
Payment payment = PaymentFactory.create();

payment.pay();
```

客户端在使用的时候只知道 `Payment`，而不知道具体的实现类是什么，这就为在运行时替换具体实现创造了条件。

这种工厂方法的优点，是 `Java`中 `SPI`的基础。在 `JDK`中最常见的例子就是 `java.sql.Driver`，接口定义如下：

```java
public interface Driver {

    Connection connect(...);
}
```

在 `JDK`编写的时候只知道 `Driver`接口，不知道未来会出现什么具体的实现，比如：

- `MySQL Driver`
- `Oracle Driver`
- `PostgreSQL Driver`
- `SQL Server Driver`

这些驱动的具体实现是由数据库厂商来做的，在 `JDK`中这些驱动实现并不存在。

`JDK`中使用 `Connection conn = DriverManager.getConnection(url);`来动态的发现实现类， `DriverManager`是一个工厂， `getConnection()`是一个静态工厂方法，返回的是一个 `Connection`接口，实际在运行时返回的对象可能是 `com.mysql.cj.jdbc.ConnectionImpl`或者 `oracle.jdbc.driver.OracleConnection`，但这些并不影响 `DriverManager.getConnection()`的设计。

服务提供者框架（Service Provider Framework）：允许一个服务拥有多个实现，由客户端在运行时获取和使用。特点：

- 系统提供一个统一的接口（服务接口）
- 允许多个服务提供者实现该接口
- 提供机制让客户端获取或选择某个实现
- 提供注册机制让服务提供者可以动态加入新的实现

服务提供者框架其实就是一种插件化架构，我们自己写接口和框架；第三方或者其他模块提供实现；客户端通过统一的 `API`获取实现，但是客户端不需要知道具体类。

服务提供者框架三个核心组件：

- 服务接口（Service Interface）：定义服务功能（定义抽象功能），让具体的服务提供者来实现这个接口。客户端需要依赖这个接口，不会依赖这个接口的具体实现类。比如： `java.sql.Connection`。
- 提供者注册API（Provider Registration API）：具体的服务提供者注册自己的实现到框架中。比如： `DriverManager.registerDriver(Driver driver)`。
- 服务访问API（Service Access API）：客户端获取服务接口的具体实现的实例。比如： `DriverManager.getConnection(url)`。

服务访问API是个静态工厂，它返回的是服务接口类型，而不是具体的实现类。客户端无需依赖具体类，也无需提前知道实现类，它可以返回默认实现，或者遍历所有可用实现。比如 `DriverManager.getConnection(url)`的代码如下：

```java
public class DriverManager {

    // 服务访问 API
    public static Connection getConnection(String url)
            throws SQLException {
        // 1. 查找所有注册的 Driver
        for (Driver d : registeredDrivers) {
            Connection conn = d.connect(url);
            if (conn != null) {
                return conn; // 返回接口类型实例
            }
        }
        throw new SQLException("No suitable driver");
    }
}
```

这段代码中：

- `getConnection()`方法就是服务访问API。
- 返回的类型是服务接口类型 `Connection`。

提供者注册API，让服务提供者自己加入到框架中，比如： `DriverManager.registerDriver(new com.mysql.cj.jdbc.Driver());`。

服务提供者框架还有一个可选的组件：服务提供者接口（Service Provider Interface），也就是我们熟知的SPI。它通常是一个工厂接口，具体服务提供者需要实现该接口，用来创建一个服务接口的实例。比如 `java.sql.Driver`：

```java
// SPI
public interface Driver {
    Connection connect(String url) throws SQLException;
}
```

服务提供者实现这个SPI：

```java
public class MySQLDriver implements Driver {
    @Override
    public Connection connect(String url) {
        return new MySQLConnection(url);
    }
}
```

框架调用：

```java
ServiceLoader<Driver> loader = ServiceLoader.load(Driver.class);
for (Driver driver : loader) {
    Connection conn = driver.connect(url);
}
```

SPI是可选的，如果没有SPI，框架会使用反射来实例化实现类；如果有SPI，框架就通过SPI来创建实例。

静态工厂方法的缺点：

1. 通常静态工厂方法所在类的构造器都是 `private` 的，无法继承。
2. 程序员很难发现这些静态工厂方法。

第二点中很难发现的意思是，使用这些静态工厂方法需要了解每个方法的用途。比如有一个类中包含静态工厂方法：

```java
public class User {

    private User() {
    }

    public static User of(String name) {
        ...
    }
}
```

这里的构造器是私有的，因此不能通过 `new User()`进行实例化，该怎么创建这个类的对象，则必须去看文档，找到对应的静态工厂方法。

静态工厂方法可以通过使用一些约定的名字，这样一看到名字就能知道是工厂方法，也可以知道是什么含义。

## 第2条：遇到多个构造器参数时要考虑使用构建器

静态工厂和构造器的局限性：不能很好的扩展到大量的可选参数。

重叠构造器（telescoping constructor）模式，这种模式中有多个构造器：

- 第一个构造器只有必要的参数；
- 第二个构造器含有一个可选参数；
- 第三个构造器含有两个可选参数；
- 以此类推；
- 最后一个构造器包含所有可选参数。

重叠构造器可行，但有很多参数的时候，客户端代码很难编写，还很难阅读。

`JavaBeans`模式，这种模式需要先调用一个无参构造器来创建对象，然后再调用 `setter`方法来设置每个必要参数以及可选参数。

`JavaBeans`模式的缺点：

- 构造过程被分到了多个调用中，在构造过程中 `JavaBean`可能处于不一致状态。
- 破会类的不可变性，需要其他方式保证线程安全。

可以使用冻结对象的方式来弥补上述缺点：当对象构造完成后，需要手动进行冻结对象，才可以使用对象。

建造者（Builder）模式，不直接生成想要的对象，而是让客户端调用 `Builder`的构造器得到一个 `Builder`对象，客户端再继续设置参数，最最后调用 `build`方法生成对象。

`Builder`模式也适用于类层次结构，也就是存在类继承体系， `Builder`也一样能用。

“使用平行层次结构的 builder 时，各自嵌套在相应的类中”，这句话是说父类和各个子类都有自己的 `Builder`。比如一个类层次：

```
Pizza
├── NyPizza
└── Calzone
```
对应的 `Builder`的层次是：

```
Pizza.Builder
├── NyPizza.Builder
└── Calzone.Builder
```

普通的 `Builder`在继承场景会出问题，比如下面的例子：

```java
// 父类
public abstract class Animal {

    private final String name;

    abstract static class Builder {

        private String name;

        public Builder name(String name) {
            this.name = name;
            return this;
        }

        abstract Animal build();
    }

    Animal(Builder builder) {
        this.name = builder.name;
    }
}
```

```java
// 子类
public class Dog extends Animal {

    private final String breed;

    public static class Builder extends Animal.Builder {

        private String breed;

        public Builder breed(String breed) {
            this.breed = breed;
            return this;
        }

        @Override
        Dog build() {
            return new Dog(this);
        }
    }

    private Dog(Builder builder) {
        super(builder);
        this.breed = builder.breed;
    }
}
```

```java
// 客户端调用
public class Client {

    public static void main() {
        Dog.Builder builder = new Dog.Builder()
                .name("旺财")
                .breed("哈士奇")
                ;
    }
}
```

这里客户端调用代码会编译失败，原因是 `name()`返回的是 `Animal.Builder`而不是 `Dog.Builder`。

解决上面问题的方案是使用一种高级的泛型技巧，叫做自引用泛型。代码如下：

```java
// 父类
public abstract class Animal {

    private final String name;

    abstract static class Builder<T extends Builder<T>> {

        private String name;

        public T name(String name) {
            this.name = name;
            return self();
        }

        protected abstract T self();

        abstract Animal build();
    }

    Animal(Builder<?> builder) {
        this.name = builder.name;
    }
}
```

```java
// 子类
public class Dog extends Animal {

    private final String breed;

    public static class Builder extends Animal.Builder<Builder> {

        private String breed;

        public Builder breed(String breed) {
            this.breed = breed;
            return this;
        }

        @Override
        protected Builder self() {
            return this;
        }

        @Override
        Dog build() {
            return new Dog(this);
        }
    }

    private Dog(Builder builder) {
        super(builder);
        this.breed = builder.breed;
    }
}
```

```java
// 客户端调用
public class Client {

    public static void main(String[] args) {
        Dog.Builder builder = new Dog.Builder()
                .name("旺财")
                .breed("哈士奇")
                ;
    }
}
```

这里的 `T extends Builder<T>`是核心设计， `Dog.Builder`定义的时候，使用 `Builder extends Animal.Builder<Builder>`将 `T`指定成 `Dog.Builder`，后续客户端使用 `name()`的时候返回的就是 `Dog.Builder`。

这种设计可以保持在继承体系中更好的使用 `Builder`，实际开发中的应用：`Lombok`的 `@SuperBuilder`是为继承场景设计的 `Builder`。

递归类型参数： `Builder<T extends Builder<T>>`，这里定义 `Builder`的时候，又引用了自己： `Builder<T>`。

很多语言有 `Self`、 `ThisType`、 `self`这样的类型，意思是当前对象所属的真实类型，比如 `Dog`调用父类的方法时需要返回 `Dog`而不是 `Animal`。 `Java`没有这种语法，不能这样写： `public Self name()`，只能通过 `Builder<T extends Builder<T>>`来模拟，所以这种叫做模拟的 `self`类型。实际项目中的应用：

- `Lombok`的 `@SuperBuilder`
- `Netty`的 `Bootstrap`
- `Hibernate`的 `CriteriaBuilder`
- `Spring Security`的 `HttpSecurity`

协变返回类型（Covariant Return Type）是 `Java 5`引入的一个语言特性，使用方式如下：

```java
// 父类
class Animal {

    Animal copy() {
        return new Animal();
    }
}
```

```java
// 子类
class Dog extends Animal {

    // 可以返回子类型Dog
    @Override
    Dog copy() {
        return new Dog();
    }

    void bark() {

    }
}
```

```java
// 客户端使用
public class Client {

    public static void main(String[] args) {
        Dog dog = new Dog();

        // 返回类型是Dog
        Dog copy = dog.copy();

        // 可以直接调用Dog独有的方法
        copy.bark();
    }
}
```

代码中父类的 `cpoy()`方法返回的类型是 `Animal`，子类重写了父类的 `copy()`方法，返回类型是 `Dog`，返回类型变了，由于 `Dog`是 `Animal`的子类，因此编译器允许，这是协变返回类型。

`Java 5`之前是不允许这种协变返回类型的，上述代码在 `Java 5`之前会报错，只能写成如下的方式：

```java
// 子类
class Dog extends Animal {

    // 不可以返回子类型Dog，只能返回父类类型Animal
    Animal copy() {
        return new Dog();
    }

    void bark() {

    }
}
```

想要调用子类特有的方法，需要进行强制类型转换： `((Dog) animal).bark();`。

## 第3条：用私有构造器或者枚举类型强化Singleton属性

单例类的对象通常代表一个无状态的对象，这个类没有成员变量（没有状态），因此这个类的对象创建1000个和创建1个对象的效果一样，故系统中只需要共享一个实例即可。

也适合表示系统中唯一的组件，比如：

- 配置中心： `ConfigManager`
- 日志管理器： `LogManager`
- `JVM Runtime`

单例会让测试变难，不能去Mock一个单例对象。

实现单例的第一种方法，使用静态成员变量：

```java
public class Singleton {

    public static final Singleton INSTANCE = new Singleton();

    private Singleton() {
        System.out.println("Singleton对象创建");
    }

    public void doSomething() {
        System.out.println("执行业务逻辑");
    }
}
```

有特权的客户端可以使用 `AccessibleObject.setAccessible`方法，通过反射机制调用私有构造器。

实现单例的第二种方法，使用静态工厂方法：

```java
public class Singleton {

    private static final Singleton INSTANCE = new Singleton();

    private Singleton() {
        System.out.println("Singleton对象创建");
    }

    public static Singleton getInstance() {
        return INSTANCE;
    }

    public void doSomething() {
        System.out.println("执行业务逻辑");
    }
}
```

也会有上面的问题：有特权的客户端可以使用 `AccessibleObject.setAccessible`方法，通过反射机制调用私有构造器。

使用第一种方式优点是：

- 很清晰很直接，看到 `Singleton.INSTANCE`就能知道这时唯一的实例。第二种方式使用静态工厂方法，使用的时候看到的只是一个普通方法名，需要看方法实现才知道是不是返回的唯一实例。
- 实现和使用更简单。第二种方式用的是方法，也就是增加了一个 `API`，形式比第一种更复杂了一些。

使用第二种方式优点是：

- 更多的灵活性，由于是一个方法，故方法中可以做更多的事情。
- 可以实现泛型单例工厂。
- 可以通过方法引用作为提供者使用。

“泛型单例工厂（Generic Singleton Factory）”。泛型只存在于编译期，运行期会进行类型擦除，当某个对象对于所有类型参数都具有完全相同的行为时，就可以只创建一个实例，然后把它安全的复用给所有泛型类型。

普通单例：

```java
public class Singleton {

    private static final Singleton INSTANCE = new Singleton();

    private Singleton() {}

    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

使用：

```java
public class Client {

    public static void main(String[] args) {
        Singleton s1 = Singleton.getInstance();
        Singleton s2 = Singleton.getInstance();

        System.out.println(s1 == s2);
    }
}

```

假设需要有一个恒等函数： `f(x) = x`，输入什么就返回什么，

- `String s = identity("hello");`返回 `hello`
- `Integer n = identity(100);`返回 `100`

该恒等函数接口定义如下：

```java
@FunctionalInterface
public interface UnaryOperator<T> {

    T apply(T value);
}
```

而如果通过最简单的实现方式来做，可能会像如下的每个类型实现一个：

```java
// String版本
public class StringIdentityFactory {

    private static final UnaryOperator<String> IDENTITY_FUNCTION = value -> value;

    public static UnaryOperator<String> identityFunction() {
        return IDENTITY_FUNCTION;
    }
}

// Integer版本
public class IntegerIdentityFactory {

    private static final UnaryOperator<Integer> IDENTITY_FUNCTION = value -> value;

    public static UnaryOperator<Integer> identityFunction() {
        return IDENTITY_FUNCTION;
    }
}
```

使用：

```java
public class Client {

    public static void main(String[] args) {
        UnaryOperator<String> stringOperator = StringIdentityFactory.identityFunction();

        UnaryOperator<Integer> integerOperator = IntegerIdentityFactory.identityFunction();

        System.out.println(stringOperator.apply("Hello"));

        System.out.println(integerOperator.apply(100));
    }
}
```

在上面两个实现类 `StringIdentityFactory`和 `IntegerIdentityFactory`中，唯一的区别只有 `String`和 `Integer`不一样，除此之外方法的实现 `value -> value`完全一样。也就是创建了两个对象，但是行为都一样，如果像这样的类型还有很多很多，那使用的时候也就会创建更多的对象，但是行为都是一样的。

可以使用泛型进行改进，代码如下：

```java
public class IdentityFactory {

    public static <T> UnaryOperator<T> identityFunction() {
        return new UnaryOperator<T>() {
            @Override
            public T apply(T value) {
                return value;
            }
        };
    }
}
```

使用：

```java
public class Client {

    public static void main(String[] args) {

        UnaryOperator<String> op1 = IdentityFactory.identityFunction();

        UnaryOperator<Integer> op2 = IdentityFactory.identityFunction();

        System.out.println(op1.apply("Hello"));

        System.out.println(op2.apply(100));

        UnaryOperator<String> op3 = IdentityFactory.identityFunction();
        System.out.println(op3.apply("Hello"));
        System.out.println(op1 == op3);
    }
}
```

这时候就不需要 `StringIdentityFactory`和 `IntegerIdentityFactory`这些类了，只需要一个 `IdentityFactory`即可。但这时候还有一个问题，上面使用中 `op1 == op3`输出为 `false`，这是因为每次调用 `IdentityFactory.identityFunction()`的时候都会生成一个对象。

上面所说的问题本质上还是没有解决，每次调用都会生成不同的对象，更好的方案是使用泛型单例工厂，只生成一个 `UnaryOperator<Object>`对象就行了。代码如下：

```java
public class IdentityFactory {

    private static final UnaryOperator<Object> IDENTITY_FUNCTION = value -> value;

    @SuppressWarnings("unchecked")
    public static <T> UnaryOperator<T> identityFunction() {
        return (UnaryOperator<T>) IDENTITY_FUNCTION;
    }
}
```

使用：

```java
public class Client {

    public static void main(String[] args) {

        UnaryOperator<String> op1 = IdentityFactory.identityFunction();

        UnaryOperator<Integer> op2 = IdentityFactory.identityFunction();

        System.out.println(op1.apply("Hello"));

        System.out.println(op2.apply(100));

        UnaryOperator<String> op3 = IdentityFactory.identityFunction();
        System.out.println(op3.apply("Hello"));
        System.out.println(op1 == op3);
    }
}
```

“通过方法引用作为提供者”，也就是可以使用如下方式： `Supplier<Singleton> supplier = Singleton::getInstance;`。意思是静态工厂方法不仅能创建对象，它本身还可以被当成一个”对象创建函数“传递给别人。

提供者 `Provider`，是一个负责创建对象的东西，可看作是帮忙创建对象的工厂，比如：

- `User createUser()`这个方法就可以看作是 `User`的 `Provider`，它负责产生 `User`对象。
- `Connection getConnection()`方法可看作是 `Connection`的 `Provider`，它负责产生数据库连接。

在 `Java 8`之前想要传递一个 `Provider`，写法优点复杂。假设有一个缓存类：

```java
public class Cache<T> {

    private T value;

    public T getOrCreate(Creator<T> creator) {
        if (value == null) {
            value = creator.create();
        }

        return value;
    }
}
```

`User`和 `Creator`定义：

```java
public class User {
}

public interface Creator<T> {

    T create();
}
```

使用：

```java
public class Client {

    public static void main(String[] args) {
        Cache<User> cache = new Cache<>();
        
        cache.getOrCreate(new Creator<User>() {
            @Override
            public User create() {
                return new User();
            }
        });
    }
}
```

上面的 `Creator`就是提供者，在使用的有点繁琐。

提供者在 `Java 8`中对应的是 `Supplier`，接口定义如下：

```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

意思非常简单：给我一个对象。比如：

- `Supplier<User>`表示一个能够产生 `User`的东西
- `Supplier<Connection>`表示一个能够产生 `Connection`的东西

使用 `Lambda`写法就是：

```java
Supplier<User> supplier = () -> new User();
```

调用这个 `Supplier`：

```java
User user = supplier.get();
```

这就相当于是 `new User();`。

方法引用语法： `ClassName::methodName`，比如上面的 `Supplier<User> supplier = () -> new User();`使用方法引用的方式就是： `Supplier<User> supplier = User::new;`。

使用静态工厂方法的单例，在实现序列化的时候需要所有实例域都是 `transient`的，还需要提供一个 `readResolve`方法，用来保证反序列化的时候不会创建多个实例。

实现 `Singleton`的第三种方法是使用枚举类型，代码如下：

```java
public enum Singleton {

    INSTANCE;

    public void doSomething() {
        System.out.println("执行业务逻辑");
    }
}
```

这种方式与第一种方式类似，但更加简洁。 `Java`中枚举在 `JVM`内部保证了实例的全局单例特性，因此无需额外的方式去保证反序列化的安全。

## 第4条：通过私有构造器强化不可实例化的能力

像一些工具类，不想被实例化，可以使用私有构造器。

## 第5条：优先考虑依赖注入来引用资源

当创建一个新的实例时，就将依赖的资源传到构造器中，这是依赖注入（Dependency Injection）的一种形式：构造器注入（Constructor Injection）。

构造器注入的优点：

- 保证依赖完整，比如 `public SpellChecker(Dictionary dictionary) {}`不传 `dictionary`时直接编译失败。
- 可以使用 `final`让依赖不可变， `private final Dictionary dictionary;`
- 更容易测试， `SpellChecker checker = new SpellChecker(new MockDictionary());`，其他代码不需要修改。

依赖注入可适用于：

- 构造器： `SpellChecker checker = new SpellChecker(dictionary);`，适合依赖比较少的情况。
- 静态工厂： `SpellChecker checker = SpellChecker.of(dictionary);`，适合需要隐藏实现的情况。
- 构建器： `SpellChecker checker = new SpellChecker.Builder().dictionary(new Dictionary()).build();`，适合参数特别多的情况。

依赖注入还可以将资源工厂传给构造器，意思是依赖注入不仅可以注入对象，还可以注入创建对象的方法。资源工厂可以重复的创建对象，一般使用工厂方法，工厂方法代码如下：

```java
public interface ShapeFactory {
    Shape create();
}
```

实现：

```java
public class CircleFactory implements ShapeFactory {

    @Override
    public Shape create() {
        return new Circle();
    }
}
```

使用：

```java
ShapeFactory factory = new CircleFactory();
Shape shape = factory.create();
```

在 `Java 8`中可使用 `Supplier<T>`来简化上面代码，无需自己定义工厂接口和工厂类，直接使用即可：

```java
Supplier<Shape> factory = Circle::new;
Shape shape = factory.get();
```

`Supplier<T>`本身表示能产生对象的东西，和工厂的定义一致。


使用 `Supplier<T>`时，如果有继承关系，则类型参数需要使用有限制的通配符类型，比如： `Supplier<? extends Animal>`。如果不使用这种方式，会像如下的代码一样出错：

```java
class Animal {
}

class Dog extends Animal {
}
```


```java
public Zoo(Supplier<Animal> supplier) {
}
```

使用： 

```java
Supplier<Dog> dogFactory = Dog::new;

Zoo zoo = new Zoo(dogFactory);
```

这里的 `new Zoo(dogFactory)`会编译出错，因为 `Zoo`需要的是 `Supplier<Animal>`，而这里给的是 `Supplier<Dog>`。泛型默认它们两个之间没有继承关系，即 `Dog`是 `Animal`的子类，但是不代表 `Supplier<Dog>`是 `Supplier<Animal>`的子类。

解决方案：

```java
public Zoo(Supplier<? extends Animal> supplier) {
}
```

## 第6条：避免创建不必要的对象

## 第7条：消除过期的对象引用

“只有当所要的缓存项的生命周期是由该键的外部引用而不是由值决定
时， WeakHashMap 才有用处”：

假设有个缓存结构： `Map<Employee, EmployeeMetadata>`， `key`是 `Employee`对象，当这个对象不存在的时候，缓存中对应的 `EmployeeMetadata`也就不再需要了，这类数据的生命周期由 `key`决定，这是 `WeakHashMap`适用的场景。

对于普通的 `HashMap`： `Map<K, V> map = new HashMap<>();`， `key`是强引用，只要 `map`存在， `key`永远不会被回收。而对于 `WeakHashMap`： `Map<K, V> map = new WeakHashMap<>();`中 `key`使用的是弱引用 `WeakReference`，假设 `Object key = new Object();`放入到缓存中 `map.put(key, value);`，之后将 `key = null`释放掉 `key`对应的对象，此时外部已经没有对这个对象的引用，只剩下 `WeakHashMap`中的弱引用，下一次 `GC`的时候 `key`将被回收，同时 `key`对应的整个 `Entry`也会自动删除，此时缓存中这一项的 `key`和 `value`都被清除掉了。

监听器（Listener），比如： 

```java
button.addActionListener(
    event -> System.out.println("clicked")
);
```

中的 `event -> ...`就是一个监听器，当按钮被点击时， `Button`通知监听器执行对应的逻辑。

回调（Callback），和监听器非常接近，比如：

```java
public interface DownloadCallback {
    void onComplete();
}
```

```java
public class Downloader {

    public void download(DownloadCallback callback) {
        // 下载完成
        callback.onComplete();
    }
}
```

使用：

```java
downloader.download(
    () -> System.out.println("下载完成")
);
```

这里的 `() -> ...`就是回调。

任何的注册机制，比如回调或监听器都有内存泄漏的风险，注册之后内部集合会保存强引用，如果处理不当会导致这些对象无法被 `GC`。解决方案：

- 显式注销，比如： `unregister(listener);`
- 使用弱引用，比如： `WeakReference<Listener>`

## 第8条：避免使用终结方法和清除方法

避免使用`finalize()`方法：

- 不可靠，在不同 `JVM`上执行的策略可能不一样，比如何时执行、是否执行都可能不一样
- 性能差
- 容易出 `Bug`
- 容易被攻击利用

从`Java 9`开始 `finalize()`方法被废弃，在 `Java 9`引入了 `java.lang.ref.Cleaner`，但是仍旧不建议使用，只是比 `finalize()`方法稍微安全了一点，但是问题依然很多，比如什么时候执行、是否执行也都是不确定的。

`finalize()`方法还有安全问题，比如有如下的类：

```java
public class SecureSession {

    private final String token;

    public SecureSession(String token) {
        if (!isValid(token)) {
            throw new SecurityException();
        }

        this.token = token;
    }

    public void transferMoney() {
        System.out.println("转账成功");
    }
    
	// ...
}
```

使用如下的方式可以绕过验证：

```java
public class EvilSession extends SecureSession {

    static EvilSession saved;

    public EvilSession() {
        super(null);
    }

    @Override
    protected void finalize() {
        saved = this;
    }
}
```

这里将没有完全构造的类记录在了一个静态域中： `static EvilSession saved;`，导致这个对象不能被回收，后续可以继续调用这个对象的方法。比如如下代码：

```java
public class Client {

    public static void main(String[] args) throws InterruptedException {
        EvilSession session = null;

        try {
            session = new EvilSession();
        } catch (Exception e) {
        }

        System.gc();

        Thread.sleep(1000);

        System.out.println(EvilSession.saved);
        EvilSession.saved.transferMoney();
    }
}
```

解决方案：

- 可以让类变成 `final`的，禁止继承
- 也可以提供一个空的并且是 `final`的 `finalize()`方法，禁止子类覆盖

在序列化的时候也会出现这样的问题。


## 第9条：try-with-resources优先于try-finally

# 第3章 对于所有对象都通用的方法

## 第10条：覆盖equals时请遵守通用约定

`equals`方法实现了等价关系，属性如下：

- 自反性： 对任何非 `null`的引用值 `x`， `x.equals(x)`必须返回 `true`。
- 对称性： 对任何非 `null`的引用值 `x`和 `y`，当且仅当 `y.equals(x)`返回 `true`时， `x.equals(y)`必须返回 `true`。
- 传递性： 对任何非 `null`的引用值 `x, y, z`，如果 `x.equals(y)`返回 `true`，且 `y.equals(z)`也返回 `true`，那么 `x.equals(z)`也必须返回 `true`。
- 一致性： 对任何非 `null`的引用值 `x, y`，只要 `equals`的比较操作在对象中所用的信息没有被修改，多次调用 `x.equals(y)`就会一致的返回 `true`。
- 对任何非 `null`的引用值 `x`， `x.equals(null)`必须返回 `false`。

## 第11条：覆盖equals时总要覆盖hashCode

## 第12条：始终要覆盖toString

## 第13条：谨慎地覆盖clone

## 第14条：考虑实现Comparable接口

# 第4章 类和接口

## 第15条：使类和成员的可访问性最小化

信息隐藏，就是只暴露必须暴露的东西，隐藏实现细节。访问控制决定谁有权限访问谁。

实体：类、接口、字段、方法、构造器等等。

访问权限不仅由访问修饰符决定，还与声明位置有关。

四种访问级别：

- `private`
- `package-private`
- `protected`
- `public`

顶层类和接口，直接定义在文件中，没有包含在其他类里面。比如下面的两个分别是顶层类和顶层接口。

顶层类：

```java
public class User {
}
```

`User`类定义在 `User.java`文件中。

顶层接口：

```java
public interface UserService {
}
```

`UserService`接口定义在 `UserService.java`文件中。

下面示例中 `Builder`不是顶层类，是嵌套类：

```java
public class User {

    static class Builder {
    }
}
```

顶层类和接口只有两种访问级别：

- `public`
- `default`（ `package-private`）

没有 `private`和 `protected`级别，修饰符其实是在描述一个类和外部类的关系，由于顶层类根本不存在外部类，故没有 `private`和 `protected`。

成员包括四类：

- 域（ `Field`），即成员变量
- 方法（ `Method`）
- 嵌套类（ `Nested Class`）
- 嵌套接口（ `Nested Interface`）

成员有四种访问级别：

- `private`： 在声明该成员的顶层类的内部才可以访问这个成员。
- `package-private`： 没有指定修饰符的时候，默认是这个级别，同一个包的可以访问
- `protected`： 声明该成员的类的子类可以访问这个成员，并且声明该成员的包内部的任何类也可以访问这个成员
- `public`： 在任何地方都可以访问该成员

访问权限可以理解成允许哪些代码依赖你，权限越大，依赖你的代码越多。

`Java 9`新增了模块系统，附带的新增了两种隐式访问级别。

## 第16条：要在公有类而非公有域中使用访问方法

## 第17条：使可变性最小化

## 第18条：复合优先于继承

