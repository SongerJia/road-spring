## 设计模式
总览->工厂模式->单例模式->代理模式->模版方法->观察者->策略->适配器->其他->高频面试题
### 总览
```
// 面试题："Spring 里用了哪些设计模式？"

// 常用回答（8 个）：
// ① 工厂模式：BeanFactory / FactoryBean
// ② 单例模式：Bean 默认单例
// ③ 代理模式：AOP 动态代理
// ④ 模板方法：JdbcTemplate、RestTemplate
// ⑤ 观察者模式：事件监听
// ⑥ 策略模式：HandlerMapping/HandlerAdapter
// ⑦ 适配器模式：HandlerAdapter
// ⑧ 装饰器模式：BeanWrapper
```
### 工厂模式
```
// ① 简单工厂 + 工厂方法：BeanFactory
// 根据 BeanDefinition 创建不同类型的 Bean

// BeanFactory.getBean() 就是工厂方法：
Object bean = beanFactory.getBean("userService");
// 具体创建逻辑在 AbstractAutowireCapableBeanFactory.createBean

// ② 抽象工厂：FactoryBean
// 复杂对象的创建交给 FactoryBean

public class UserFactoryBean implements FactoryBean<User> {
    @Override
    public User getObject() {
        // 自定义创建逻辑
        return new User("张三", 25);
    }

    @Override
    public Class<?> getObjectType() {
        return User.class;
    }
}

// 注入时拿到的是 getObject() 返回的对象
// 加 & 前缀拿到 FactoryBean 本身：&userFactoryBean

// ③ 静态工厂：<bean factory-method="">
// ④ 实例工厂：<bean factory-bean="" factory-method="">
```

|对比|BeanFactory|FactoryBean|
|---|---|---|
|**类型**|容器接口|创建 Bean 的接口|
|**作用**|管理所有 Bean|复杂对象创建|
|**getBean 结果**|Bean 实例|getObject() 结果|
### 单例模式
```
// Spring 的 Bean 默认单例（singleton）

// 特点：
// ① 每个 Bean 名一个实例（缓存 singletonObjects）
// ② 线程安全取决于 Bean 本身（无状态安全）
// ③ 容器管理生命周期

// 和标准单例的区别：
// 标准单例：类自身保证（private 构造器）
// Spring 单例：容器保证（每次 getBean 返回同一个）

// 单例注意：
// ① 无状态 Bean 安全（Service、Dao）
// ② 有状态 Bean 要小心并发（如注入有状态的字段）
// ③ 尽量无状态（不存实例字段）
```
### 代理模式
```
// 动态代理：JDK Proxy / CGLIB

// 场景：
// ① AOP 切面（事务、日志）
// ② @Async、@Cacheable
// ③ MyBatis Mapper 接口（JDK 代理实现）

// 例子：MyBatis Mapper
// 接口没有实现类，MyBatis 用 JDK 代理生成实现
// 调用方法 → 代理 → 执行 SQL → 返回结果

// 静态代理 vs 动态代理：
// 静态：手写代理类（编译期）
// 动态：运行时生成（Spring AOP 用动态）
```
### 模板方法
```
// 模板方法：父类定义流程骨架，子类实现细节

// 典型：JdbcTemplate
public class JdbcTemplate {
    // 模板方法：定义数据库操作流程
    public <T> T execute(StatementCallback<T> action) {
        // ① 获取连接
        Connection con = getConnection();
        // ② 创建 Statement
        Statement stmt = con.createStatement();
        // ③ 执行业务（子类/回调实现）
        T result = action.doInStatement(stmt);
        // ④ 关闭资源
        closeStatement(stmt);
        closeConnection(con);
        return result;
    }
}

// 好处：
// ① 公共流程统一（连接、事务、关闭）
// ② 业务只写核心逻辑
// ③ 避免重复代码

// 其他模板：RestTemplate、RedisTemplate、MongoTemplate
```
### 观察者模式
```
// Spring 事件：发布-订阅

// ① 定义事件
public class OrderCreatedEvent extends ApplicationEvent {
    private final Long orderId;

    public OrderCreatedEvent(Object source, Long orderId) {
        super(source);
        this.orderId = orderId;
    }

    public Long getOrderId() {
        return orderId;
    }
}

// ② 发布事件
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher publisher;

    public void createOrder() {
        // 业务...
        publisher.publishEvent(new OrderCreatedEvent(this, orderId));
    }
}

// ③ 监听事件（解耦！）
@Component
public class OrderEventListener {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // 发短信（异步）
        // 更新统计
        // 日志
    }
}

// 好处：
// ① 解耦（发布者不知道谁监听）
// ② 同步/异步（@Async + @EventListener）
// ③ 扩展性（加监听器不改源码）
```
### 策略模式+适配器模式
策略模式（HandlerMapping/HandlerAdapter）
```
// ① HandlerMapping：多个实现，选择策略
// RequestMappingHandlerMapping：注解映射
// SimpleUrlHandlerMapping：URL 映射
// BeanNameUrlHandlerMapping：Bean 名映射

// ② 依赖注入时的多实现 + @Primary/@Qualifier 也是策略选择

// ③ @ConditionalOnXxx 按条件选择实现（自动配置）
```
适配器模式（HandlerAdapter）
```
// HandlerAdapter 就是适配器：
// 不同的 Handler 类型 → 统一的调用方式

// 场景：
// ① @RequestMapping 方法 → RequestMappingHandlerAdapter
// ② HttpRequestHandler → HttpRequestHandlerAdapter
// ③ Controller 接口 → SimpleControllerHandlerAdapter

// 好处：
// DispatcherServlet 统一调用 handle()
// 具体差异由适配器处理

// 另一个例子：
// 不同的存储访问 → 统一接口：
// JdbcTemplate / JpaTemplate / MongoTemplate
// 通过适配消除差异
```
### 其他设计模式
```
// ① 装饰器模式：
// BeanWrapper（包装 Bean 属性）
// 多个 BeanPostProcessor 层层包装

// ② 责任链模式：
// 拦截器链（HandlerInterceptor）
// 过滤器链（Filter）
// AOP 通知链

// ③ 建造者模式：
// BeanDefinitionBuilder
// UriComponentsBuilder

// ④ 组合模式：
// CompositeCacheManager（缓存管理器组合）

// ⑤ 门面模式：
// JdbcTemplate 封装了底层 JDBC 的复杂
// ApplicationContext 封装了 BeanFactory 的复杂

// ⑥ 原型模式：
// prototype 作用域的 Bean
```
### 面试高频题
#### 题目1：Spring用了哪些设计模式
```
// 工厂：BeanFactory
// 单例：默认 Bean 作用域
// 代理：AOP
// 模板方法：JdbcTemplate
// 观察者：事件监听
// 策略：HandlerMapping
// 适配器：HandlerAdapter
// 装饰器：BeanWrapper
// 责任链：拦截器
```
#### 题目2：BeanFactory和FactoryBean的区别
```
// BeanFactory：容器（getBean）
// FactoryBean：创建复杂对象的工厂
// getBean 返回 getObject() 结果，&前缀拿工厂本身
```
#### 题目3：事件机制怎么用
```
// ApplicationEvent + publishEvent + @EventListener
// 发布订阅解耦
```
#### HandlerAdapter为什么是适配器
```
// 统一不同 Handler 的调用方式
// DispatcherServlet 不用关心具体类型
```
