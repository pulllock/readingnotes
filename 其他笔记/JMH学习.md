## Mode
Mode表示使用的模式：

- Throughput：整体吞吐量，比如1秒内可以执行多少次调用。
- AverageTime：调用平均时间，比如每次调用平均耗时。
- SampleTime：随机取样，最后输出取样结果的分布，比如99%的调用在xxx毫秒内，99.99%的调用在xxx毫秒内
- SingleShotTime：上面模式都是默认一次iteration是1s，而SingleShotTime只运行一次，往往把warmup次数设为0，用于测试冷启动时的性能
- All：所有的benchmark的模式

## Iteration
Iteration是JMH最小测试单位，大部分模式下，一次iteration代表一秒，JMH在这一秒内不断调用需要benchmark的方法，然后根据模式对其采样，计算吞吐量或者平均执行时间等。

## Warmup
Warmup指的是在benchmark前进行的预热行为。因为 JVM 的 JIT 机制的存在，如果某个函数被调用多次之后，JVM 会尝试将其编译成为机器码从而提高执行速度，为了让 benchmark 的结果更加接近真实情况就需要进行预热。

参数iterations表示预热的次数。
## @Benchmark
`@Benchmark`注解表示方法需要进行benchmark，跟JUnit的`@Test`类似。

## @BenchmarkMode
`@BenchmarkMode`注解表示使用的模式。

## @State
`@State`表明某个类是一个状态，接收一 个Scope参数来表示该状态的共享范围。测试方法可能接收参数，需要提供单个的参数类，`@State`注解定义了给定类实例的可用范围。

Scope主要有一下几种：

- Thread：该状态为每个线程共享。
- Benchmark：该状态在运行相同测试的所有线程间共享。
- Group：实例分配给每个线程组。

## @OutputTimeUnit
`@OutputTimeUnit`注解表示结果所用的时间单位。

## 启动选项Options

- include：benchmark所在类的名字，这里使用的是正则表达式对所有类进行匹配。
- fork：进行fork的次数，fork为2的话，JMH会fork出两个进程进行测试。
- warmupIterations：预热的迭代次数。
- measurementIterations：实际测量的迭代次数。

## @Param
`@Param`注解用来指定某项参数的多种情况，适合用来测试一个方法在不同参数输入的情况下的性能。

## @Setup
`@Setup`注解主要用来初始化，在执行benchmark之前被执行。 

## @TearDown
`@TearDown`注解主要用来资源的回收，在执行benchmark结束之 后执行。

## @CompilerControl
`@CompilerControl`注解可以控制compiler的行为，比如强制inline，不允许编译等。

## @Group
`@Group`注解可以把多个benchmark定义为同一个group，它们会被同时执行，主要用于测试多个相互之间存在影响的方法。

## @Level
`@Level`注解用于控制`@Setup`和`@TearDown`的调用时机，默认是Level.Trial，即benchmark开始前和结束后。

- Level.Trial，默认的level，在全部benchmark运行的开始前和结束后。
- Level.Iteration：在一次迭代开始前和结束后。
- Level.Invocation：在每个方法调用之前和之后，不推荐使用。

## @Profiler
`@Profiler`注解可以让JMH支持一 些profiler，可以显示等待时间和运行时间比，热点函数等。

## @Measurement
`@Measurement`度量，一些基本的测试参数，参数iterations表示进行测试的轮次，time表示每轮进行的时长，timeUnit表示时长单位。

## @Threads
`@Threads`测试线程数，根据具体情况而定。

## @OutputTimeUnit
`@OutputTimeUnit`测试结果的时间类型。

## @Fork
`@Fork`需要运行的测试数量，即迭代的次数。每个测试运行在单独的JVM进程中，也可以指定JVM参数。

## @Warmup

`@Warmup`注解表示预热。
