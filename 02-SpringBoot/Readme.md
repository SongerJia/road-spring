> **模块边界**：Spring 核心机制（IoC / AOP / 事务）→ Spring 模块；微服务治理（注册中心 / 网关 / 熔断）→ Spring Cloud 模块；具体数据访问框架（MyBatis / JPA）细节 → 各自模块。本模块覆盖 Boot 的自动配置、配置体系、Web、Actuator、启动与部署。

**① Spring Boot 总览**

1. 是什么、解决的痛点
2. 与 Spring 的关系
3. 核心特性
4. 版本演进（2.7 弃用 spring.factories、3.0 移除；3.0 = Java 17 + Jakarta EE 9+）
5. 项目结构约定与多模块工程结构
6. DevTools 与 Spring Boot CLI

**② 自动配置** 7. @SpringBootApplication 组合注解 8. @EnableAutoConfiguration 原理 9. 自动配置注册：spring.factories → AutoConfiguration.imports 10. 条件注解 @Conditional 系列 11. @ConditionalOnMissingBean 与 Bean 覆盖规则 12. @ConditionalOnProperty / @Profile 配置开关 13. 自动配置的顺序依赖（@AutoConfigureBefore / After / Order） 14. 自动配置的排除与调试、启动失败分析 15. 自定义 starter

**③ 配置** 16. application.yml / properties 17. 配置优先级与加载顺序 18. ConfigData API（2.4+） 19. 多环境配置与激活方式 20. @ConfigurationProperties vs @Value 21. @ConfigurationProperties 的构造器绑定、@ConfigurationPropertiesScan 与校验 22. 松散绑定（Relaxed Binding）的多命名风格 23. 配置元数据（spring-configuration-metadata.json） 24. 配置加密（了解） 25. 外部化配置

**④ 起步依赖与依赖管理** 26. starter 机制 27. spring-boot-dependencies BOM 与 parent 28. 常用 starter

**⑤ 内嵌容器** 29. Tomcat / Jetty / Undertow 30. 内嵌容器启动原理 31. 可执行 jar / fat jar 的结构（BOOT-INF、JarLauncher、URLStreamHandler）

**⑥ Web 开发** 32. REST 开发 33. 静态资源处理 34. 错误处理与 Problem Details（RFC 7807） 35. 拦截器与 Filter 注册 36. WebMvcConfigurer 37. 跨域 38. 统一异常处理 39. 参数校验 @Valid 40. 文件上传 41. 健康探针（liveness / readiness）

**⑦ 常用能力** 42. 日志 43. JSON（Jackson）配置 44. 缓存 45. 异步与定时任务 46. 数据访问 starter

**⑧ Actuator 与可观测性** 47. Actuator 端点 48. 自定义端点 49. 健康检查 50. Micrometer 指标与 Micrometer Tracing（替代 Sleuth）

**⑨ 启动流程与优化** 51. 启动三阶段（准备环境 → 创建容器 → 刷新容器）与事件插入点 52. ApplicationContextInitializer 53. ApplicationRunner / CommandLineRunner 54. 启动性能优化（延迟初始化）与 AOT / Native Image（GraalVM）

**⑩ 测试** 55. @SpringBootTest 56. MockMvc 57. @MockBean 与切片测试 58. Testcontainers 与 Docker Compose 支持

**⑪ 打包部署与常见坑** 59. 打包插件 60. 容器化 61. 优雅停机（2.3+ graceful，含超时） 62. 常见坑清单（自动配置未生效 / Bean 被抢占 / 配置不生效 / jar 内资源读取 / 主类找不到）