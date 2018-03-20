# 注解

@Test， 表明该方法是一个测试方法。

@BeforeClass，测试用例初始化时执行，方法必须是static的，方法在类被加载的时候就调用，那时候还没创建测试对象实例。

@AfterClass，所有测试用例执行完毕之后，可以进行收尾工作。方法必须是static的。

@Before，每个测试方法执行前都要执行一次。

@After，每个测试方法执行后都要执行一次。

@Test(expected = xxx.class)，expected属性验证是否抛出期望的异常，expected属性的值是一个异常的类型，如果抛出了期望的异常，测试通过，否则不通过。

@Test(timeouit = xxx)，传入一个毫秒时间给测试方法，如果测试方法在指定的时间内没有运行完，测试用例失败。

@Ignore，测试方法会被忽略，可以给该注解传递一个String参数，用来说明为什么忽略测试方法。

# 常用断言

assertEquals()

assertFalse()

assertTrue()

assertNull()

assertNotNull()

assertSame()

assertNotSame()

assertThat(value, matcher statement)需要结合hamcrest使用。