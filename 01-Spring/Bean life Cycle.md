## Bean生命周期
全貌总览->实例化->属性填充->Aware->BeanPostProcessor->初始化->销毁->源码对应->面试高频题
### 生命周期全貌总览
完整生命周期图
```
创建 Bean
    │
    ├── ① 实例化（new，反射构造器）
    │
    ├── ② 属性填充（依赖注入 populateBean）
    │
    ├── ③ Aware 回调
    │    ├── BeanNameAware（名字）
    │    ├── BeanFactoryAware（工厂）
    │    └── ApplicationContextAware（上下文）
    │
    ├── ④ BeanPostProcessor#postProcessBeforeInitialization（初始化前）
    │
    ├── ⑤ 初始化
    │    ├── InitializingBean#afterPropertiesSet（接口）
    │    └── @PostConstruct / init-method（注解/配置）
    │
    ├── ⑥ BeanPostProcessor#postProcessAfterInitialization（初始化后）★ AOP 代理在这里
    │
    ├── ...使用中...
    │
    └── ⑦ 销毁
         ├── @PreDestroy / destroy-method
         └── DisposableBean#destroy
```
### 实例化
createBeanInstance
```
// AbstractAutowireCapableBeanFactory.createBeanInstance
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
    // ① 使用工厂方法创建
    // <bean factory-method="createInstance"/>
    if (mbd.getFactoryMethodName() != null) {
        return instantiateUsingFactoryMethod(...);
    }

    // ② 自动装配构造器（@Autowired 构造器）
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(...);
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR) {
        return autowireConstructor(beanName, mbd, ctors, null);  // 构造器注入
    }

    // ③ 默认无参构造器
    return instantiateBean(beanName, mbd);  // 反射 newInstance
}
```
三种实例化方式
```
// ① 工厂方法：静态工厂/实例工厂
// ② 构造器注入：@Autowired 构造器
// ③ 默认构造器：无参 new（最常见）
```
### 属性填充
populateBean
```
// AbstractAutowireCapableBeanFactory.populateBean
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {

    // ① 处理 InstantiationAwareBeanPostProcessor
    // 有机会修改属性值 / 提前结束注入
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                return;  // 返回 false 跳过后续注入
            }
        }
    }

    // ② 按类型收集需要注入的属性
    PropertyValues pvs = mbd.getPropertyValues();

    // ③ 按名称/类型自动装配（@Autowired）
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME ||
        mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
        ...
    }

    // ④ 执行属性注入（重点！）
    // 这里有 AutowiredAnnotationBeanPostProcessor
    // 扫描 @Autowired 字段/方法 → 反射 set → 注入
    applyPropertyValues(...);
}
```
@Autowired注入
```
// AutowiredAnnotationBeanPostProcessor
// 是 BeanPostProcessor，在 postProcessProperties 中处理注入

public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
    // ① 扫描类中带 @Autowired 的字段和方法
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);

    // ② 执行注入
    metadata.inject(bean, beanName, pvs);
    return pvs;
}

// 注入逻辑（字段注入）：
// AutowiredFieldElement.inject()
// ① 解析依赖类型（byType）
// ② 从容器获取：beanFactory.resolveDependency()
//    - 多个候选 → @Qualifier/@Primary 判断
//    - 处理延迟注入（@Lazy）
// ③ 反射设置字段值
Field field = ...
field.setAccessible(true);
field.set(bean, resolvedDependency);  // 反射注入
```
### Aware回调
Aware接口体系
```
// Aware：让 Bean 感知容器的"回调接口"
public interface Aware {}

// 常见 Aware：
// BeanNameAware：setBeanName(String name)
// BeanFactoryAware：setBeanFactory(BeanFactory)
// ApplicationContextAware：setApplicationContext(ApplicationContext)
// EnvironmentAware：setEnvironment(Environment)
// ResourceLoaderAware：setResourceLoader(ResourceLoader)
// MessageSourceAware：setMessageSource(MessageSource)
```
源码处理
```
// AbstractAutowireCapableBeanFactory.invokeAwareMethods
private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
        // ① BeanNameAware
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        // ② BeanClassLoaderAware
        if (bean instanceof BeanClassLoaderAware) {
            ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
        }
        // ③ BeanFactoryAware
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(this);
        }
    }
}

// 注意：
// ApplicationContextAware 不在上面处理
// 在 ApplicationContextAwareProcessor（BeanPostProcessor）中处理
// 因为 ApplicationContextAware 需要 ApplicationContext（比 BeanFactory 高一级）
```
应用场景
```
// ① 需要获取容器/其他 Bean 时：
public class MyBean implements ApplicationContextAware {
    private ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext context) {
        this.context = context;
    }

    public void doSomething() {
        UserService userService = context.getBean(UserService.class);
    }
}

// ② 不推荐直接使用（侵入性），一般用注入替代
```
### BeanPostProcessor
```
// BeanPostProcessor：Bean 初始化的"前后拦截器"
// 每个 Bean 创建时都会经过它

public interface BeanPostProcessor {
    // 初始化之前
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    // 初始化之后 ★ AOP 代理在这里！
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```
内置的重要BeanPostProcessor
```
// ① AutowiredAnnotationBeanPostProcessor
//    处理 @Autowired 注入（上面的属性填充）

// ② CommonAnnotationBeanPostProcessor
//    处理 @PostConstruct / @PreDestroy

// ③ ApplicationContextAwareProcessor
//    处理 Aware 回调

// ④ AnnotationAwareAspectJAutoProxyCreator ★
//    AOP 的核心！postProcessAfterInitialization 中创建代理
//    Spring 的事务、@Async 等全靠它
```
自定义BeanPostProcessor
```
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // 所有 Bean 初始化前都会经过这里
        System.out.println("初始化前: " + beanName);
        return bean;  // 必须返回！
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // 初始化后：可以在这里包装代理
        System.out.println("初始化后: " + beanName);
        return bean;
    }
}

// 注意：
// ① 必须返回 bean（否则丢失）
// ② 可以对特定类处理（instanceof 判断）
// ③ 返回代理对象 = AOP 的原理
```
### 面试高频

> **Q:** "BeanPostProcessor 的作用？AOP 在哪里生效？" 
> **A:** "BeanPostProcessor 是 Bean 初始化的前后拦截器。AOP 在 postProcessAfterInitialization 生效：AnnotationAwareAspectJAutoProxyCreator 检查 Bean 是否有切面匹配，匹配则返回 JDK 代理或 CGLIB 代理对象。所以 AOP 代理发生在 Bean 初始化之后。"
### 初始化
```
// 三种初始化方式（优先级顺序）：
// ① @PostConstruct（注解，最高优先级）
// ② InitializingBean#afterPropertiesSet（接口）
// ③ init-method 属性（XML/配置，最低优先级）

// 源码：
protected void invokeInitMethods(String beanName, RootBeanDefinition mbd, Object bean) {
    // ① InitializingBean 接口
    if (bean instanceof InitializingBean) {
        ((InitializingBean) bean).afterPropertiesSet();
    }

    // ② 自定义 init-method
    String initMethodName = mbd.getInitMethodName();
    if (initMethodName != null) {
        invokeCustomInitMethod(beanName, bean, mbd);
    }
}

// 顺序：@PostConstruct → afterPropertiesSet → init-method
// @PostConstruct 由 CommonAnnotationBeanPostProcessor
// 在 postProcessBeforeInitialization 中执行（所以在最前面）

@Component
public class MyBean implements InitializingBean {
    @PostConstruct
    public void postConstruct() {
        System.out.println("① @PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("② InitializingBean");
    }

    @PostConstruct
    // ③ init-method 用 XML/@Bean(initMethod=...)
    public void customInit() {
        System.out.println("③ init-method");
    }
}
```
### 销毁
```
// 三种销毁方式（和初始化对称）：
// ① @PreDestroy（注解）
// ② DisposableBean#destroy（接口）
// ③ destroy-method（配置）

// 触发时机：
// ① 容器关闭（close()）
// ② 单例 Bean 才会执行销毁
// ③ prototype 不管理销毁（容器不管）

// 源码：registerDisposableBeanIfNecessary
// ① 容器关闭时遍历销毁
// ② 先销毁依赖它的 Bean（先创建的后销毁）
```
### 完整源码对照表
| 生命周期阶段 | 源码位置                            | 说明             |
| ------ | ------------------------------- | -------------- |
| 实例化    | createBeanInstance              | 反射创建           |
| 属性填充   | populateBean                    | 依赖注入           |
| Aware  | invokeAwareMethods              | 回调             |
| 初始化前   | postProcessBeforeInitialization | @PostConstruct |
| 初始化    | invokeInitMethods               | 接口/配置          |
| 初始化后   | postProcessAfterInitialization  | AOP 代理         |
| 销毁     | destroyBean                     | 容器关闭           |
