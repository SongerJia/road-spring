## IOC容器
IOC是什么->容器体系->BeanFactory vs ApplicationContext->refresh流程->核心源码->面试高频题
### IOC是什么
IOC Inversion of control 控制反转：对象的创建和管理权从自己new反转给容器。DI（依赖注入）是IOC的实现方式。
对比
```
// ❌ 传统：自己 new（控制权在自己）
public class OrderService {
    private UserService userService = new UserService();  // 自己创建
    private StockService stockService = new StockService();
}

// ✅ IOC：容器创建并注入（控制权反转）
@Service
public class OrderService {
    @Autowired
    private UserService userService;   // 容器注入
    @Autowired
    private StockService stockService;
}

// 好处：
// ① 解耦（不依赖具体实现，面向接口）
// ② 统一管理（单例、生命周期）
// ③ 方便测试（换 Mock 实现）
```
### 容器体系
核心接口继承体系
```
BeanFactory（顶层接口：getBean）
    ↓
ListableBeanFactory（可列举：getBeansOfType）
    ↓
HierarchicalBeanFactory（分层：父子容器）
    ↓
ApplicationContext（应用上下文）
    ├── ConfigurableApplicationContext（refresh）
    │   ├── ClassPathXmlApplicationContext
    │   └── AnnotationConfigApplicationContext  ← 现代常用
    ├── WebApplicationContext
    └── ...
```
关键接口职责
```
// ① BeanFactory：容器最核心的接口
public interface BeanFactory {
    Object getBean(String name);           // 按名称获取
    <T> T getBean(Class<T> requiredType);  // 按类型获取
    boolean isSingleton(String name);
    // ...
}

// ② ApplicationContext 在 BeanFactory 基础上扩展：
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory,
        HierarchicalBeanFactory, MessageSource, ApplicationEventPublisher, ResourcePatternResolver {
    // ① 国际化（MessageSource）
    // ② 事件发布（ApplicationEventPublisher）
    // ③ 环境配置（EnvironmentCapable）
    // ④ 资源加载（ResourcePatternResolver）
}

// 结论：
// ApplicationContext = BeanFactory + 企业级功能
```
BeanFactory vs ApplicationContext

|对比|BeanFactory|ApplicationContext|
|---|---|---|
|**懒加载**|默认懒加载|默认启动时预实例化|
|**功能**|基础 getBean|国际化/事件/资源|
|**实现**|DefaultListableBeanFactory|AnnotationConfigApplicationContext|
|**使用**|内部底层|实际开发|
|**关系**|ApplicationContext 内部持有 BeanFactory|扩展了它|
### refresh流程
refresh是容器的核心入口
```
// AbstractApplicationContext.refresh()
// 容器创建的核心流程，面试必背
public void refresh() throws BeansException {
    synchronized (this.startupShutdownMonitor) {

        // ① prepareRefresh：准备刷新
        // 记录启动时间、设置状态标志、初始化属性源
        prepareRefresh();

        // ② obtainFreshBeanFactory：获取 BeanFactory
        // 创建/刷新 BeanFactory（解析配置 → 注册 BeanDefinition）
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // ③ prepareBeanFactory：准备 BeanFactory
        // 设置类加载器、SpEL 解析器、注册内置 BeanPostProcessor
        prepareBeanFactory(beanFactory);

        // ④ postProcessBeanFactory：BeanFactory 后置处理（子类扩展）
        postProcessBeanFactory(beanFactory);

        // ⑤ invokeBeanFactoryPostProcessors：执行 BeanFactoryPostProcessor
        // 处理 @Configuration 类、扫描包注册 BeanDefinition
        invokeBeanFactoryPostProcessors(beanFactory);

        // ⑥ registerBeanPostProcessors：注册 BeanPostProcessor
        // 注册所有 BeanPostProcessor（处理 Bean 生命周期）
        registerBeanPostProcessors(beanFactory);

        // ⑦ initMessageSource：初始化国际化
        initMessageSource();

        // ⑧ initApplicationEventMulticaster：初始化事件广播器
        initApplicationEventMulticaster();

        // ⑨ onRefresh：留给子类的钩子（创建 Web 容器等）
        onRefresh();

        // ⑩ registerListeners：注册监听器
        registerListeners();

        // ⑪ finishBeanFactoryInitialization：实例化所有单例 Bean
        // 重点！非懒加载的单例在这里创建
        finishBeanFactoryInitialization(beanFactory);

        // ⑫ finishRefresh：完成刷新
        // 发布 ContextRefreshedEvent 事件
        finishRefresh();
    }
}
```
refresh流程总结
```
① 准备
② 创建 BeanFactory
③ 准备 BeanFactory
④ 子类扩展
⑤ 执行 BeanFactoryPostProcessor（扫描注册）
⑥ 注册 BeanPostProcessor（生命周期钩子）
⑦ 国际化
⑧ 事件广播器
⑨ 子类钩子
⑩ 注册监听器
⑪ 实例化所有单例 Bean ← 重点！
⑫ 完成，发布事件
```
面试回答模版
```
面试官："Spring 容器的启动流程？"

"核心是 refresh() 方法：
① prepareRefresh 准备
② obtainFreshBeanFactory 创建 BeanFactory（解析配置注册 BeanDefinition）
③ prepareBeanFactory 准备（类加载器、SpEL、内置处理器）
⑤ invokeBeanFactoryPostProcessors 执行后置处理器（扫描 @Component、处理 @Configuration）
⑥ registerBeanPostProcessors 注册 Bean 后置处理器
⑦⑧ 初始化国际化、事件广播器
⑪ finishBeanFactoryInitialization 实例化所有单例 Bean ← 关键
⑫ 完成，发布 ContextRefreshedEvent

其中 ⑪ 是最核心的：遍历 BeanDefinition，逐个实例化、注入依赖。"
```
### BeanDefinition
```
// BeanDefinition：Bean 的"元数据"描述
// 记录：类名、作用域、是否懒加载、初始化方法、依赖等
// 是创建 Bean 的"蓝图"

public interface BeanDefinition {
    String getBeanClassName();      // 类名
    String getScope();              // 作用域（singleton/prototype）
    boolean isLazyInit();           // 是否懒加载
    boolean isSingleton();          // 是否单例
    String getInitMethodName();     // 初始化方法
    String getDestroyMethodName();  // 销毁方法
    String[] getDependsOn();        // 依赖的 Bean
```
BeanDefinition的来源
```
// ① 配置文件：
// <bean id="userService" class="...UserService"/> → BeanDefinition

// ② 注解扫描：
// @Component/@Service/@Repository → ClassPathBeanDefinitionScanner
// 扫描包 → 找到注解 → 注册 BeanDefinition

// ③ @Bean 方法：
// @Configuration + @Bean → ConfigurationClassPostProcessor
// 解析配置类 → 方法转为 BeanDefinition

// ④ 编程式：
// BeanDefinitionBuilder 手动构建
```
关键存储结构
```
// BeanDefinition 存在哪？
// DefaultListableBeanFactory：
// private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
// 名字 → BeanDefinition

// 所以：IOC 容器本质上 = BeanDefinition 注册表 + 单例缓存
```
### Bean流程
getBean流程
```
// AbstractBeanFactory.getBean(name)
public Object getBean(String name) {
    return doGetBean(name, null, null, false);
}

protected <T> T doGetBean(...) {
    // ① 先从单例缓存取
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        // 缓存命中 → 处理 FactoryBean → 返回
        bean = getObjectForBeanInstance(sharedInstance, ...);
    } else {
        // ② 缓存没有 → 创建
        // 处理父子容器：父容器有就交给父容器
        // ③ 检查依赖（dependsOn）先创建依赖
        // ④ 创建 Bean：
        //    - 单例：createBean → 缓存
        //    - 原型：createBean（不缓存）
        //    - 其他作用域
    }
    return bean;
}
```
createBean流程
```
// AbstractAutowireCapableBeanFactory.createBean
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) {
    // ① 解析 Bean 的 class
    // ② 处理 override 方法（lookup-method 等）
    // ③ 实例化前置处理（BeanPostProcessor 有机会返回代理）
    Object bean = resolveBeforeInstantiation(...);
    if (bean != null) return bean;

    // ④ 核心：创建 Bean 实例
    Object beanInstance = doCreateBean(beanName, mbd, args);
    return beanInstance;
}
```
doCreateBean流程
```
protected Object doCreateBean(...) {
    // ① 创建实例（反射 new）
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);

    // ② 提前暴露（解决循环依赖！三级缓存）
    addSingletonFactory(beanName, () -> getEarlyBeanReference(...));

    // ③ 属性填充（依赖注入！）
    populateBean(beanName, mbd, instanceWrapper);

    // ④ 初始化（Aware → 前置处理 → init → 后置处理）
    exposedObject = initializeBean(beanName, exposedObject, mbd);

    // ⑤ 注册销毁方法
    registerDisposableBeanIfNecessary(beanName, bean, mbd);
    return exposedObject;
}
```
### 面试高频题
#### 题目1：什么是IOC，有什么好处
```
// 控制反转：对象创建和管理交给容器
// 好处：解耦、统一管理、方便测试
```
#### 题目2：BeanFactory和ApplicationContext的区别
```
// BeanFactory：基础容器（懒加载）
// ApplicationContext：扩展（预实例化+国际化+事件+资源）
```
#### 题目3：refresh流程
```
// 12 步，重点：⑪ 实例化所有单例 Bean
```
#### 题目4：BeanDefinition是什么
```
// Bean 的元数据（蓝图）
// 类名、作用域、懒加载、初始化方法
// 来源：XML/注解扫描/@Bean
```
#### 题目5：容器启动时做了什么
```
// 解析配置 → 注册 BeanDefinition → 
// 执行后置处理器 → 实例化单例 Bean → 发布事件
```
#### 题目6：getBean流程
```
// 先查单例缓存 → 没有则创建（实例化→填充→初始化）
// 创建后放入缓存
```
