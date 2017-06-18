# ProxyFactoryBean

## FactoryBean会调用getObject方法

### 初始化通知器链initializeAdvisorChain，将配置的通知器添加到拦截器列表中去
### 生成单例代理对象getSingletonInstance

#### 刷新TargetSource freshTargetSource
#### 获取要代理的接口getTargetClass
#### 设置代理对象的接口setInterfaces
#### 创建AOP代理createAopProxy

##### 获取AOP代理工厂getAopProxyFactory
##### 创建AOP代理createAopProxy

###### 接口类使用JdkDynamicAopProxy
###### 非接口使用CglibProxyFactory.createCglibProxy

#### 获取AOP代理getProxy
##### 使用具体的子类来获取代理aopProxy.getProxy
###### 使用JdkDynamicAopProxy来获取aop代理
###### 使用CglibAopProxy来获取aop代理

### 生成原型代理对象newPrototypeInstance
