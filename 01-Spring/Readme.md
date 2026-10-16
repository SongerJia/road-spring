> **模块边界**：Spring Boot（自动配置 / starter / 内嵌容器）与 Spring Cloud（微服务）→ 各自模块；Spring MVC 与 Servlet 容器的细节已并入本模块；响应式（WebFlux / Reactor）→ 单独方向。本模块覆盖 IoC / AOP / 事务 / MVC / 扩展机制。

**① Spring 总览**

1. Spring 是什么、解决的问题
2. Spring 生态与模块组成
3. IoC 与 DI 概念
4. Spring 的优缺点
5. 版本基线（Spring 6 / Boot 3：Java 17+、Jakarta EE 9+、AOT）

**② IoC 容器** 6. BeanFactory vs ApplicationContext 7. 容器启动流程与 `refresh()` 具体步骤 8. BeanDefinition 与 Bean 的三种定义方式（XML / 注解 / 编程式） 9. 依赖注入方式（构造器 / setter / 字段） 10. @Autowired / @Resource / @Qualifier 11. 注解式装配（@Component / @Service / @Repository / @Controller） 12. @Configuration：full vs lite 与 CGLIB 增强 13. @Bean / @Import / @DependsOn / @Order 14. 组件扫描 @ComponentScan 15. 属性注入：@Value / @PropertySource 16. Bean 作用域 17. Bean 生命周期、Aware 接口、初始化 / 销毁回调 18. BeanPostProcessor / BeanFactoryPostProcessor 19. 循环依赖与三级缓存（三级各自存什么、为何要 ObjectFactory） 20. 循环依赖的边界（构造器注入 / prototype 无法解决） 21. 条件装配 @Conditional 22. FactoryBean 23. 容器父子关系与关闭钩子

**③ AOP** 24. AOP 概念 25. 动态代理（JDK vs CGLIB 的精确选择规则） 26. @Aspect 与通知类型 27. 切点表达式 28. 代理失效场景 29. AOP 应用

**④ 事务** 30. 编程式 vs 声明式 31. @Transactional 原理与属性（rollbackFor / timeout / readOnly / propagation） 32. 传播行为（REQUIRED / REQUIRES_NEW / NESTED 及 savepoint） 33. 事务隔离级别 34. 事务失效场景（自调用不走代理 / 非 public / 异常被吞 / 多线程等）

**⑤ 资源、事件与环境** 35. Resource 抽象 36. 事件机制与异步事件 37. Environment 与 Profile 38. 国际化

**⑥ Spring MVC** 39. DispatcherServlet 与请求流程 40. 核心组件（HandlerMapping / HandlerAdapter） 41. 常用注解与参数绑定 42. 自定义参数解析器与返回值处理器 43. HttpMessageConverter 与内容协商 44. RESTful 支持 45. 返回值处理与视图解析 46. 拦截器 47. 异常处理 48. 数据校验 49. CORS 跨域 50. Filter / Listener 与 Servlet 容器 51. 过滤器 vs 拦截器 vs AOP

**⑦ 扩展与横切能力** 52. Spring 中的设计模式 53. Spring 扩展点总览（BPP / BFPP / Aware / InitializingBean / ImportSelector / ApplicationListener） 54. 类型转换与 SpEL 55. 缓存抽象 @Cacheable 及原理（代理 + CacheManager） 56. 异步 @Async 及原理（代理 + TaskExecutor） 57. Spring 测试支持（@ContextConfiguration / MockMvc）

**⑧ 高频辨析收口** 58. BeanFactory vs FactoryBean 59. ApplicationContext vs BeanFactory 60. @Autowired vs @Resource 61. @Component vs @Bean 62. @Configuration full vs lite 63. 单例 Bean 与线程安全 64. 边界：响应式 Spring WebFlux → 单独方向