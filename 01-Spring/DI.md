## 依赖注入与循环依赖
三种注入方式->注入方式对比->@Autowried解析流程->循环依赖问题->三级缓存->为什么能解决->为什么构造器注入不行->面试高频题
### 三种注入方式
构造器注入
```
@RestController
public class OrderController {
    private final OrderService orderService;  // final！推荐

    // 构造器注入（Spring 4.3+ 单个构造器可省略 @Autowired）
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }
}
```
Setter注入
```
@Service
public class OrderService {
    private StockService stockService;

    @Autowired
    public void setStockService(StockService stockService) {
        this.stockService = stockService;
    }
}
```
字段注入
```
@Service
public class OrderService {
    @Autowired
    private StockService stockService;  // 最常用但最不推荐
}
```
### 注入方式对比
| 对比              | 构造器     | Setter | 字段    |
| --------------- | ------- | ------ | ----- |
| **不可变性**        | ✅ final | ❌      | ❌     |
| **必填依赖**        | ✅ 强制    | ❌ 可选   | ❌ 可选  |
| **循环依赖**        | ❌ 不支持   | ✅ 支持   | ✅ 支持  |
| **测试友好**        | ✅ 好     | 中      | ❌ 难   |
| **Spring 官方推荐** | ✅ 推荐    | 备选     | ❌ 不推荐 |
```
// Spring 官方推荐：构造器注入
// 理由：
// ① 依赖不可变（final）
// ② 保证依赖非空（创建时必须提供）
// ③ 方便单元测试（直接 new + 传参）
// ④ 避免循环依赖（构造器循环直接报错，暴露问题）

// 但实际项目：
// 字段注入最多（@Autowired 简单）
// 阿里规范：强制要求构造器或 setter（避免字段注入）
```
### @Autowried 解析流程
resolveDependency
```
// DefaultListableBeanFactory.resolveDependency
// @Autowired 注入时的核心解析方法

public Object resolveDependency(DependencyDescriptor descriptor,
        @Nullable String requestingBeanName, ...) {

    // ① 处理 @Lazy 延迟注入（返回代理）
    Object value = getAutowireCandidateResolver()
        .getLazyResolutionProxyIfNecessary(descriptor, requestingBeanName);
    if (value != null) return value;

    // ② 按类型查找候选 Bean
    Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);

    // ③ 多个候选 → 处理歧义
    if (matchingBeans.isEmpty()) {
        if (isRequired(descriptor)) {
            throw new NoSuchBeanDefinitionException(...);  // 没有 → 报错
        }
        return null;
    }

    if (matchingBeans.size() > 1) {
        // 多个候选：
        // ① @Primary 优先
        // ② @Priority 排序
        // ③ 参数名/字段名匹配（@Qualifier）
        // ④ 都不行 → NoUniqueBeanDefinitionException
        String autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
        ...
    }
    return instance;
}
```
多个候选的处理顺序
```
// 接口有多个实现时的注入规则：
// ① @Primary：标记优先实现
@Service
@Primary
public class DefaultUserService implements UserService {}

// ② @Qualifier：指定名称
@Autowired
@Qualifier("vipUserService")
private UserService userService;

// ③ 字段名匹配（兜底）
@Autowired
private UserService vipUserService;  // 字段名 = Bean 名

// ④ @Resource：JDK 注解，按名称优先
@Resource(name = "vipUserService")
private UserService userService;
```
常见问题
```
// 报错：expected single matching bean but found 2
// 原因：接口多个实现，没指定
// 解决：@Primary / @Qualifier

// 报错：No qualifying bean of type
// 原因：没有该类型的 Bean（忘了 @Service/扫描不到）
```
### 循环依赖的问题
什么是循环依赖
```
// A 依赖 B，B 依赖 A（互相依赖）
@Service
public class A {
    @Autowired
    private B b;   // A 需要 B
}

@Service
public class B {
    @Autowired
    private A a;   // B 需要 A
}

// 创建 A：需要 B → 创建 B：需要 A → 死循环！
```
没有三级缓存的后果
```
// 场景：
// ① 创建 A（实例化）→ 需要注入 B
// ② 创建 B（实例化）→ 需要注入 A
// ③ A 还没创建完（没有放入缓存）→ 报错？

// 报错：BeanCurrentlyInCreationException
// "Requested bean is currently in creation: Is there an unresolvable circular reference?"
```
### 三级缓存
三级缓存结构
```
// DefaultSingletonBeanRegistry 中的三个 Map：

// 一级缓存：成品 Bean（创建完成的单例）
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

// 二级缓存：早期暴露的 Bean（创建中，未完成初始化）
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

// 三级缓存：ObjectFactory（能生成早期引用的工厂）
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```
三级缓存的作用

|缓存|存什么|用途|
|---|---|---|
|**一级**|完成创建的 Bean|正常获取|
|**二级**|早期 Bean（未初始化）|解决循环依赖时直接给半成品|
|**三级**|ObjectFactory 工厂|延迟生成早期引用（支持 AOP）|
为什么需要三级而不是二级
```
// 核心：AOP 代理！
// 如果 Bean 需要 AOP 代理：
// ① 三级缓存存 ObjectFactory：
//    getEarlyBeanReference() → 判断是否需要代理 → 返回代理或原对象
// ② 如果只有二级缓存（直接存实例）：
//    无法在"需要时才判断 AOP"→ 要么提前代理（浪费），要么无法代理（出错）

// 所以：
// 三级缓存 = 延迟 AOP 代理的决策
// 在循环依赖时，B 拿到的 A 应该是 A 的代理（如果 A 有 AOP）
```
### 循环依赖解决流程
```
// 场景：A 依赖 B，B 依赖 A，都用字段注入

// ① 创建 A：
//    doCreateBean(A)
//    → createBeanInstance(A) 实例化
//    → addSingletonFactory(A的工厂)  ← 提前暴露到三级缓存！
//    → populateBean(A) 填充属性：需要 B

// ② 获取 B：
//    getBean(B) → 缓存没有 → 创建 B
//    doCreateBean(B)
//    → createBeanInstance(B) 实例化
//    → addSingletonFactory(B的工厂)  ← 提前暴露
//    → populateBean(B)：需要 A

// ③ 获取 A（循环！）：
//    getSingleton(A)：
//    一级缓存没有
//    二级缓存没有
//    三级缓存有 A 的 ObjectFactory
//    → 调用工厂生成早期 A（可能代理）
//    → 放入二级缓存，移除三级缓存
//    → 返回 A（半成品，未初始化）
//    B 的属性填充完成 ✅

// ④ B 继续初始化完成 → 放入一级缓存

// ⑤ 回到 A：
//    A 拿到 B（已完成）→ 填充完成
//    → 初始化完成 → 放入一级缓存
//    ✅ 循环依赖解决！
```
源码关键方法
```
// ① 提前暴露（doCreateBean 中）：
protected Object doCreateBean(...) {
    ...
    // 单例 && 允许提前暴露（默认允许）
    boolean earlySingletonExposure = (mbd.isSingleton() &&
            this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));

    if (earlySingletonExposure) {
        // 把工厂放入三级缓存
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }
    ...
}

// ② getEarlyBeanReference（三级缓存生成早期引用）：
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    // 检查是否需要 AOP 代理
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {
            // AOP 代理在这里决定
            exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
        }
    }
    return exposedObject;
}

// ③ getSingleton（三级取用）：
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 一级缓存
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // 二级缓存
        singletonObject = this.earlySingletonObjects.get(beanName);
        if (singletonObject == null && allowEarlyReference) {
            // 三级缓存：取工厂，生成早期引用
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
                singletonObject = singletonFactory.getObject();
                this.earlySingletonObjects.put(beanName, singletonObject);
                this.singletonFactories.remove(beanName);
            }
        }
    }
    return singletonObject;
}
```
### 哪些循环依赖解决不了
构造器注入的循环依赖
```
@Service
public class A {
    public A(B b) {}   // 构造器需要 B
}

@Service
public class B {
    public B(A a) {}   // 构造器需要 A
}

// 结果：报错！BeanCurrentlyInCreationException
// 原因：构造器注入时 Bean 还没实例化
// → 无法提前暴露（addSingletonFactory 还没执行）
// → 永远拿不到半成品 → 死循环 → 报错

// 结论：构造器注入天然避免循环依赖（直接报错）
```
原型prototype 的循环引用
```
// 原型 Bean：不缓存、不提前暴露 → 无法解决 → 报错
// 结论：循环依赖只在 singleton 下能解决
```
@Async的循环依赖
```
// @Async 的代理在初始化后创建
// 三级缓存拿到的不是异步代理 → 可能出错
// Spring Boot 2.6+ 默认禁止循环依赖：
// spring.main.allow-circular-references=false

// 结论：最好避免循环依赖（设计问题）
```
### 面试高频题
#### 题目1：三种注入方式
```
// 构造器/setter/字段
// 官方推荐构造器（不可变、必填、好测试）
```
#### 题目2：什么是循环依赖
```
// A 依赖 B，B 依赖 A
```
#### 题目3：三级缓存是什么？为什么是三级
```
// 一级成品、二级早期、三级工厂
// 第三级是为了延迟 AOP 代理的决策
```
#### 题目4：循环依赖怎么解决
```
// ① A 实例化后提前暴露（三级缓存）
// ② B 创建时需要 A → 从三级取半成品
// ③ B 完成 → A 完成
```
#### 题目5：构造器循环依赖为什么不行
```
// 构造器注入时实例化都没完成
// 无法提前暴露 → 死循环报错
```
#### 题目6：循环依赖怎么避免
```
// 能！设计上避免：
// ① 拆解依赖（B 依赖 A 的接口方法，不依赖 Bean）
// ② 用 @Lazy 延迟注入
// ③ 抽公共依赖
// ④ Spring Boot 2.6+ 默认禁止
```
