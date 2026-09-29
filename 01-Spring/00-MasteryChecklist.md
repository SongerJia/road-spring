# 精通清单 · Spring 核心

> **使用说明**
> 1. 合上资料，录音回答每题 → 对照三级标准自评（及格 / 良好 / 精通）
> 2. 未达「精通」的题 → 回锚点材料补 → **3 天后重答**
> 3. 单题标准：**及格** = 核心结论 + 主干要点；**良好** = 及格 + 能解释「为什么」+ 一个应用/坑；**精通** = 良好 + 源码级细节 + 连环追问不卡壳
>
> **本板块通关线**：5 题全部达到「精通」档

---

## 1. IOC 容器原理？BeanFactory 与 ApplicationContext？

**锚点**：官方文档 Core 章节 / 源码 `DefaultListableBeanFactory`

- [ ] **及格**：IOC = 控制反转（对象创建与管理交给容器）；`BeanFactory` 是底层容器，`ApplicationContext` 是它的**增强子接口**（多了事件、国际化、AOP 集成等）
- [ ] **良好**：及格 + 能说出容器工作流程——解析配置（XML/注解）→ 注册 `BeanDefinition` → 实例化 → 依赖注入 → 放入**单例池**（`singletonObjects` Map）；`@Component`/`@Bean` 如何变成 BeanDefinition
- [ ] **精通**：良好 + 能应对追问——**BeanFactory vs FactoryBean** 的区别（工厂 vs 工厂生产的对象）、**循环依赖**怎么解决（三级缓存）、`@Scope` 的作用域（singleton/prototype/request/session）、懒加载 `@Lazy`

**自评记录**：___（档位） 日期：___ 复答：___

---

## 2. 依赖注入的方式？循环依赖怎么解决？

**锚点**：官方文档 / 源码 `DefaultSingletonBeanRegistry`（三级缓存）

- [ ] **及格**：三种注入方式——**构造器注入 / Setter 注入 / 字段注入**（`@Autowired`）；推荐构造器注入（不可变、可测试、无循环依赖问题）
- [ ] **良好**：及格 + 能说出 `@Autowired` 查找规则（先 **byType**，有多个候选再 **byName**，都没有报错，`@Qualifier` 指定）+ `@Resource`（byName 优先）的区别
- [ ] **精通**：良好 + 能应对追问——**循环依赖**（A↔B）的解决原理：**三级缓存**（singletonObjects → earlySingletonObjects → singletonFactories），提前暴露**半成品对象**；**为什么构造器注入无法解决循环依赖**（对象还没实例化无法提前暴露）；AOP 代理对象怎么通过第三级缓存注入

**自评记录**：___（档位） 日期：___ 复答：___

---

## 3. Bean 生命周期？

**锚点**：官方文档 / 源码 `AbstractAutowireCapableBeanFactory`

- [ ] **及格**：五个阶段——**实例化 → 属性填充 → 初始化 → 使用 → 销毁**
- [ ] **良好**：及格 + 能说出初始化阶段细节——`Aware` 接口（BeanName/BeanFactory/ApplicationContext）、`@PostConstruct` / `InitializingBean` / `init-method` 的执行顺序、**BeanPostProcessor**（`postProcessBeforeInitialization` / `postProcessAfterInitialization`）夹在中间
- [ ] **精通**：良好 + 能应对追问——完整顺序默写（实例化 → 属性填充 → Aware → Before → @PostConstruct → InitializingBean → init-method → After → 使用 → @PreDestroy → DisposableBean → destroy-method）、BeanPostProcessor 与 Aware 的顺序、**循环依赖时半成品对象何时被提前暴露**（第三级缓存）、销毁在容器关闭时如何触发

**自评记录**：___（档位） 日期：___ 复答：___

---

## 4. AOP 原理？

**锚点**：官方文档 / 源码 `ProxyFactory`

- [ ] **及格**：AOP 概念——**切面 / 切点 / 通知（前置/后置/环绕/异常）/ 织入**；实现靠**动态代理**
- [ ] **良好**：及格 + 能说出两种代理——**JDK 动态代理**（基于接口，`Proxy` + `InvocationHandler`）vs **CGLIB**（基于类继承，生成子类）；**Spring AOP 基于代理，AspectJ 基于字节码织入**
- [ ] **精通**：良好 + 能应对追问——**为什么 Spring AOP 只能方法级别**（代理只能拦截方法调用，不能拦截字段访问）、**同类内部调用（`this` 调用）代理失效**的原因与解决（自注入/`AopContext`）、多个切面执行顺序（`@Order`）、AOP 与事务的关系

**自评记录**：___（档位） 日期：___ 复答：___

---

## 5. 声明式事务原理？

**锚点**：官方文档 Data Access / 源码 `TransactionInterceptor`

- [ ] **及格**：`@Transactional` 声明事务；事务管理器 `PlatformTransactionManager`；事务基于 **AOP 代理**实现
- [ ] **良好**：及格 + 能说出**传播行为**（REQUIRED / REQUIRES_NEW / NESTED / SUPPORTS 等）+ 隔离级别（与数据库隔离级别对应）+ 回滚规则（默认只回滚 RuntimeException 和 Error）
- [ ] **精通**：良好 + 能应对追问——**事务失效的场景**（同类自调用、异常被 catch、方法非 public、类未被 Spring 管理、传播行为设置错）+ 自调用失效的原因（代理不生效）+ **数据库事务与 Spring 事务的配合**（一个物理事务/多个逻辑事务）、`@Transactional` 放在实现类还是接口上

**自评记录**：___（档位） 日期：___ 复答：___

---

> **通关检查**：本板块 5 题是否全部达到「精通」档？是 → 进入下一板块；否 → 标记未达标的题号：___
