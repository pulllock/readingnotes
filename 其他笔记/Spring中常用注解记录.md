## Spring自带注解
### @Autowired
`@Autowired`自动装配，为了消除getter和setter，以及bean属性中的property。默认是类型匹配。

如果属性找不到，又不想让Spring抛异常，可以设置`@Autowired`注解的required属性设置为false。

如果一个接口有多个实现类，想用`@Autowired`注解接口，可以使用`@Qualifier("xxx")`来同时注解，xxx为接口实现类的类名。

### @Component
`@Component`注解将一个类定义为Bean，默认名称是小写开头的类名，还可以指定名称`@Component("xxx")`

### @Controller,@Service,@Repository
` @Controller`,`@Service`,`@Repository`和`@Component`注解语意一样，更加细化，`@Controller`代表展示层，`@Service`代表业务层，`@Repository`代表存储层

### @Required
`@Required`依赖检查

### @Value
`@Value`注入SpEL表达式，可以放在字段，方法，或参数上，`@Value(value="SpEL表达式")`或者`@Value(value="#{xxx.yyy}")`

### @Qualifier
`@Qualifier`限定描述符，用于细粒度选择候选者

### @Scope
`@Scope`来定义Bean的作用域

### @ImportResource
`@ImportResource`用来引用一个资源，对应一个xml文件

### @Configuration
`@Configuration`用来取代xml来配置Spring

### @Bean
`@Bean`用来标识配置个初始化一个由Spring IOC容器管理的新对象的方法，类似xml中的`bean/`标签，注解是单例的，要改变作用域可以使用`@Scope`

### @ComponentScan
`@ComponentScan`自动扫描包名下使用`@Service`，`@Resource`，`@Repository`，`@Controller`注解的类，并注册为Bean

### @Lazy
`@Lazy`表示延迟初始化

### @Primary
`@Primary`自动装配时出现多个bean候选者，被注解为`@Primary`的bean作为首选者

### @Aspect
`@Aspect`声明一个切面

### @After，@Before，@Around
`@After`，`@Before`，`@Around`声明通知器

### @Pointcut
`@Pointcut`定义拦截规则，声明切点

### @PropertySource
`@PropertySource`注入一个配置文件

### @DependsOn
`@DependsOn`定义bean初始化以及销毁时的顺序

### @Profile
`@Profile`注解在类或方法上，再不同的情况下选择实例化不同Bean

### @EnableAsync
`@EnableAsync`注解在配置类上，开启对异步任务的支持

### @Async
`@Async`注解在实际执行的Bean的方法中，声明这是一个异步任务

### @EnableScheduling
`@EnableScheduling`注解在配置类上，开启对定时任务的支持

### @Scheduled
`@Scheduled`注解在实际执行的方法上，声明这是一个计划任务，`@Scheduled(cron="0 2511 ? * *")`是linux和Unix系统下的定时任务，`@Scheduled(fixDelay=5000)`延迟5秒执行，`@Scheduled(fixedRate=5000)`每隔5秒执行一次

### @Conditional
`@Conditional`用在配置类中，根据满足某一个特定条件创建一个特定的bean

### @EnableAspectJAutoProxy
`@EnableAspectJAutoProxy`注解在配置类上，开启对AspectJ自动代理的支持

### @EnableWebMvc
`@EnableWebMvc`开启web mvc的配置支持

### @EnableConfigurationProperties
`@EnableConfigurationProperties`开启对`@ConfigurationProperties`注解的支持

### @EnableJpaRepositories
`@RnableJpaRepositories`开始对Spring Data JPA Repository的支持

### @EnableTransactionManagement
`@EnableTransactionManagent`开启注解式事务的支持

### @EnableCaching
`@EnableCaching`开启注解式缓存的支持

### @Order
`@Order`调整配置类加载顺序

### @RequestMapping
`@RequestMapping`用来映射web请求，路径和参数

### @ResponseBody
`@ResponseBody`将返回类型直接输入到http response body中，输出json格式的数据

### @RequestBody
`@RequestBody`request的参数需要放在请求体内

### @PathVariable
`@PathVariable`接收路径参数，将方法参数绑定到URI模板变量的值上

### @RestController
`@RestController`相当于`@Controller`和`@ResponseBody`的组合，创建REST类型的控制器

### @ControllerAdvice
`@ControllerAdvice`可以将对于控制器的全局配置放在同一个位置

### @ExceptionHandler
`@ExceptionHandler`用于全局处理控制器里的异常

### @InitBinder
`@InitBinder`用来设置WebDataBinder，WebDataBinder用来自动绑定前台请求参数到Model中

### @ModelAttribute
`@ModelAttribute`注解在方法上，表明该方法的目的是添加一个或者多个模型属性，会在`@RequestMapping`方法调用前被调用

### @RequestParam
`@RequestParam`将请求的参数绑定到方法中的参数上

### @HttpEntity
`@HttpEntity`能获得request请求和response响应，还能访问请求体和响应体

### @SpringBootApplication
`@SpringBootApplication`主要作用开启自动配置，相当于`@ComponentScan``@Configuration``@EnableAutoConfiguration`的组合，exclude参数可以关闭特定的自动配置

### @EnableRedisHttpSession

### @Entity
`@Entity`表明这是一个实体Bean 

### @Lookup
`@Lookup`定义一个lookup方法

### @RequestScope
`@RequestScope`表示一个组件的作用域是request

### @SessionScope
`@SessionScope`表示一个组件的作用域是session

### @ApplicationScope
`@ApplicationScope`表示一个组件的作用域是application

### @EnableLoadTimeWeaving
`@EnableLoadTimeWeaving`注册一个加载时织入

### @EventListener
`@EventListener`注册成一个事件监听器

### @NumberFormat
`@NumberFormat`格式化数字

### @DateTimeFormat
`@DateTimeFormat`格式化日期

### @Transactional
`@Transactional`注解事务

### @GetMapping
`@GetMapping`get方法和`@RequestMapping`

### @PostMapping
### @PutMapping
### @DeleteMapping
### @PatchMapping
### @RestControllerAdvice
### @RequestAttribute
### @SessionAttribute
### @CookieValue
### @RequestHeader
### @RequestPart
### @CrossOrigin
`@CrossOrigin`启用CORS

### @EnableWebSocket
`@EnableWebSocket`启用websocket支持

### @EnableWebSocketMessageBroker
`@EnableWebSocketMessageBroker`
### @SendTo
### @MessageMapping
### @JmsListener
### @EnableJms

### @Cacheable
`@Cacheable`声明式缓存

### @CacheEvict
`@CacheEvict`清除缓存

### @CachePut
`@CachePut`更新缓存

### @Caching

### @CacheConfig
`@CacheConfig`缓存的配置类

### @Priority


## Java注解
### @Resource
`@Resource`跟`@Autowired`类似，`@Resource`默认通过name属性匹配，找不到再按type匹配，`@Resource`是J2EE的注解。

### @PostConstruct，
`@PostConstruce`指定初始化方法，会在Bean初始化之后被Spring容器执行

### @PreDestory
`@PreDestroy`指定销毁方法，会在Spring容器关闭前销毁Bean的时候被执行

### @Inject
`@Inject`和`@Autowired`等价，只是没有required属性

### @Named
`@Named`指定Bean名字，对应于Spring的`@Qualifier`默认的根据Bean名字注入

### @Qualifier
`@Qualifier`只对应于Spring的`@Qualifier`中扩展限定描述符注解，只做扩展，没有value属性

### @ConstructorProperties
`@ConstructorProperties`指明构造器属性名字

## JPA注解
### @PersistenceContext
`@PersistenceContext`用于注入EntityManagerFactory

### @PersistenceUnit
`@PersistenceUnit`用于注入EntityManager
