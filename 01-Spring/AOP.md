## AOP
AOP是什么->核心概念->动态代理->JDK vs CGLIB->代理创建时机->代理链执行->事务/@Async的坑->面试高频题
### AOP是什么
Aspect Oriented Programming 面向切面编程：把横切逻辑（日志、事务、权限）从业务代码中抽取处理，动态织入。
```
// 痛点：
public class OrderService {
    public void createOrder() {
        long start = System.currentTimeMillis();
        // 业务逻辑...
        log.info("耗时: " + (System.currentTimeMillis() - start));
    }
    // 每个方法都要写日志 → 重复代码！
}

// AOP 解决：
@Around("@annotation(loggable)")
public Object log(ProceedingJoinPoint pjp) {
    long start = System.currentTimeMillis();
    Object result = pjp.proceed();  // 执行业务
    log.info("耗时: " + ...);
    return result;
}
// 日志逻辑抽出来，业务代码干净
```
### 核心概念
|                   |                           |       |
| ----------------- | ------------------------- | ----- |
| **切面 Aspect**     | 横切逻辑的集合（通知+切点）            | 一个类   |
| **切点 Pointcut**   | 匹配哪些方法                    | 正则    |
| **通知 Advice**     | 具体逻辑（before/after/around） | 方法    |
| **连接点 JoinPoint** | 被拦截的方法                    | 匹配的方法 |
| **织入 Weaving**    | 把通知应用到目标方法                | 生成代理  |
五种通知
```
@Before     // 方法执行前
@After      // 方法执行后（无论成败）
@AfterReturning  // 正常返回后
@AfterThrowing  // 抛出异常后
@Around     // 环绕（最强，可控制整个流程）
```
### 动态代理
Sprng AOP基于动态代理
```
// 两种代理方式：
// ① JDK 动态代理：基于接口（InvocationHandler）
// ② CGLIB 代理：基于子类（继承，字节码生成）

// 选择规则：
// ① 目标类有接口 → JDK 代理（默认）
// ② 没有接口 → CGLIB 代理
// ③ Spring Boot 2.x+ 默认强制 CGLIB（proxyTargetClass=true）
```
Jdk动态代理
```
// 原理：Proxy.newProxyInstance + InvocationHandler

public class JdkProxy implements InvocationHandler {
    private final Object target;

    public JdkProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("前置逻辑");
        Object result = method.invoke(target, args);  // 反射调用目标
        System.out.println("后置逻辑");
        return result;
    }

    // 创建代理：
    public static Object create(Object target) {
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),  // 必须有接口！
            new JdkProxy(target)
        );
    }
}

// 限制：目标类必须实现接口
// 否则 Proxy 无法创建代理类
```
CGLIB代理
```
// 原理：继承目标类，生成子类，覆写方法

// CGLIB 生成目标类的子类
// 子类覆写目标方法 → 调用时走代理逻辑

// 限制：
// ① 目标类不能被 final（不能继承）
// ② 目标方法不能被 final（不能覆写）
// ③ 私有方法不能被代理
```

|对比|JDK 代理|CGLIB 代理|
|---|---|---|
|**原理**|接口 + InvocationHandler|继承生成子类|
|**要求**|必须有接口|类不能 final|
|**性能**|JDK8+ 已优化|创建慢，调用快|
|**Spring Boot 默认**|❌（Boot2+ 用 CGLIB）|✅ 默认|
|**适用范围**|接口方法|所有非 final 方法|
### AOP代理创建的时机
在哪创建
```
// 回顾 Bean 生命周期：
// postProcessAfterInitialization ← AOP 代理在这里！
// AnnotationAwareAspectJAutoProxyCreator.postProcessAfterInitialization
```
流程
```
// AbstractAutoProxyCreator.postProcessAfterInitialization
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (bean != null) {
        // 检查是否需要代理
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);  // 核心
        }
    }
    return bean;
}

// wrapIfNecessary：是否需要代理
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    // ① 已经是代理 → 跳过
    // ② 没有切面 → 跳过
    // ③ 获取该 Bean 匹配的 Advisor（通知）
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);

    if (specificInterceptors != DO_NOT_PROXY) {
        // ④ 创建代理！
        Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, ...);
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;  // 返回代理，替代原 Bean
    }
    return bean;
}

// createProxy：
protected Object createProxy(...) {
    // ① 判断用 JDK 还是 CGLIB（proxyTargetClass）
    // ② 收集 Advisors
    // ③ 创建代理工厂 → 生成代理对象
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.addAdvisors(advisors);
    return proxyFactory.getProxy(classLoader);
}
```
核心结论
```
// ① AOP 代理 = Bean 初始化后，被 AnnotationAwareAspectJAutoProxyCreator 包装
// ② 容器中存的是"代理对象"（getBean 拿到的是代理）
// ③ 代理对象调用方法 → 走切面 → 反射调目标
```
### 代理链执行
代理调用流程
```
// 代理对象调用方法时：
// ① 进入 InvocationHandler（JDK）或 MethodInterceptor（CGLIB）
// ② 获取该方法的 Advisor 链（多个通知）
// ③ 按顺序执行通知链（责任链模式）
// ④ 最后调用目标方法

// CGLIB 的 MethodInterceptor：
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) {
    // ① 获取通知链
    List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

    // ② 有通知 → 递归执行通知链
    if (!chain.isEmpty()) {
        return new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy)
            .proceed();  // 链式调用
    }

    // ③ 没有通知 → 直接调用目标
    return methodProxy.invoke(target, args);
}
```
通知链执行
```
// MethodInvocation.proceed()
// ① 执行第一个拦截器（通知）
// ② 通知内部调用 proceed() → 下一个通知
// ③ 直到最后一个 → 调用目标方法

// 顺序（@Around 为例）：
// Around.before → Before → 目标方法 → AfterReturning/After → Around.after
```
### AOP的坑
#### 自调用不生效
```
@Service
public class OrderService {

    @Transactional
    public void createOrder() {
        // 事务生效（外部调用）
    }

    public void processOrder() {
        this.createOrder();  // ❌ 自调用！事务不生效！
        // 因为 this 是原始对象，不是代理对象
    }
}

// 原因：
// createOrder() 被代理拦截，是因为调用方拿的是"代理对象"
// this.createOrder() 调用的是"原始对象"的方法 → 绕过代理 → 切面失效  this永远指向当前实例本身是编译期+运行期绑定

// 解决：
// ① 注入自己（代理）
@Autowired
private OrderService orderService;  // 注入的是代理

public void processOrder() {
    orderService.createOrder();  // ✅ 走代理
}

// ② 从容器获取代理（AopContext）
public void processOrder() {
    ((OrderService) AopContext.currentProxy()).createOrder();
}
```
#### final类和方法不能代理
```
@Service
public final class UserService {   // ❌ final 类 → CGLIB 无法继承
    @Transactional
    public void update() {}
}

// CGLIB 无法生成子类 → 代理失败 → 切面失效
// 解决：去掉 final
```
#### JDK代理的类转换问题
```
// JDK 代理是接口实现，不是子类
// 强转成实现类会失败

UserService userService = (UserService) proxy;  // ✅ 接口可以
// UserServiceImpl impl = (UserServiceImpl) proxy;  // ❌ ClassCastException！
```
### 面试高频题
#### AOP的原理
```
// 动态代理：JDK（接口）或 CGLIB（子类）
// 在 Bean 初始化后包装成代理对象
// 代理调用方法 → 执行通知链 → 调用目标
```
#### JDK和CGLIB区别
```
// JDK：接口 + InvocationHandler
// CGLIB：继承 + 字节码
// Spring Boot 默认 CGLIB
```
Aop代理在什么时候创建
```
// Bean 生命周期 postProcessAfterInitialization
// AnnotationAwareAspectJAutoProxyCreator.wrapIfNecessary
```
#### 自调用为什么不生效
```
// this 是原始对象，不走代理
// 解决：注入代理 / AopContext.currentProxy
```
#### 事务怎么通过aop实现
```
// @Transactional → 事务拦截器（TransactionInterceptor）
// 是环绕通知：开启事务 → 执行方法 → 提交/回滚
```
#### spring里aop用在哪些地方
```
// ① 事务管理 @Transactional
// ② @Async 异步
// ③ @Cacheable 缓存
// ④ 自定义切面（日志、权限、限流）
```
