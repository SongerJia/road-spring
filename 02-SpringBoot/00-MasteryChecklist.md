# 精通清单 · Spring Boot

> **使用说明**
> 1. 合上资料，录音回答每题 → 对照三级标准自评（及格 / 良好 / 精通）
> 2. 未达「精通」的题 → 回锚点材料补 → **3 天后重答**
> 3. 单题标准：**及格** = 核心结论 + 主干要点；**良好** = 及格 + 能解释「为什么」+ 一个应用/坑；**精通** = 良好 + 源码级细节 + 连环追问不卡壳
>
> **本板块通关线**：4 题全部达到「精通」档

---

## 1. 自动配置原理？

**锚点**：官方文档 / 源码 `@EnableAutoConfiguration` / `AutoConfigurationImportSelector`

- [ ] **及格**：`@SpringBootApplication` 是组合注解（`@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`）；自动配置 = **按条件自动装配**常用组件的 Bean
- [ ] **良好**：及格 + 能说出机制——`AutoConfigurationImportSelector` 读取 `META-INF/spring/...AutoConfiguration.imports`（老版本 `spring.factories`）里的**自动配置类**，再通过 **`@ConditionalOnXxx`** 系列条件注解按需生效（类存在才配、Bean 缺失才配）
- [ ] **精通**：良好 + 能应对追问——怎么**排查**生效了哪些自动配置（`--debug` 看自动配置报告 / `ConditionEvaluationReport`）、`@ConditionalOnMissingBean` 的作用（**用户自定义优先**）、怎么**覆盖**自动配置（自定义 Bean / `@SpringBootApplication(exclude=...)`）、自动配置类里怎么用 `@ConfigurationProperties` 绑定配置

**自评记录**：___（档位） 日期：___ 复答：___

---

## 2. Starter 是什么？怎么自定义一个？

**锚点**：官方文档 Creating Your Own Starter

- [ ] **及格**：Starter = **依赖集合 + 自动配置**的封装；引入一个 starter 就能开箱即用（如 `spring-boot-starter-web`）
- [ ] **良好**：及格 + 能说出自定义步骤——① 建 `xxx-spring-boot-autoconfigure` 模块（写自动配置类 + 条件注解 + 注册到 imports 文件）② 建 `xxx-spring-boot-starter` 模块（只放依赖，依赖 autoconfigure）③ 引入后自动生效
- [ ] **精通**：良好 + 能应对追问——为什么 starter 能**简化依赖管理**（统一版本、传递依赖）、自动配置类为什么需要 `@ConditionalOnClass`（避免类缺失时启动失败）、`@ConfigurationProperties` + `@EnableConfigurationProperties` 怎么把配置绑定到属性类、命名规范（官方 starter 叫 `spring-boot-starter-xxx`，自定义建议 `xxx-spring-boot-starter`）

**自评记录**：___（档位） 日期：___ 复答：___

---

## 3. 配置加载顺序？@ConfigurationProperties vs @Value？

**锚点**：官方文档 Externalized Configuration

- [ ] **及格**：配置来源**优先级**——命令行参数 > 环境变量 > application-{profile}.yml > application.yml > 默认值；`@Value` 注入单个值，`@ConfigurationProperties` 批量绑定到对象
- [ ] **良好**：及格 + 能解释——**外部化配置**的价值（同一套代码不同环境不同配置）、`@ConfigurationProperties` 的优点（类型安全、批量、支持复杂结构、宽松绑定）+ 与 `@Value` 的使用场景取舍
- [ ] **精通**：良好 + 能应对追问——**宽松绑定（relaxed binding）**（`user-name` / `userName` / `USER_NAME` 都能绑到 `userName`）、`@ConfigurationProperties` 的校验（`@Validated`）、配置的**占位符与默认值**（`${server.port:8080}`）、生产配置**加密**方案（jasypt / 配置中心）、profile 激活方式（`spring.profiles.active` / 打包参数）

**自评记录**：___（档位） 日期：___ 复答：___

---

## 4. 内嵌容器（Tomcat）原理？

**锚点**：官方文档 / 源码 `ServletWebServerFactory`

- [ ] **及格**：Spring Boot **内嵌 Tomcat**（也可切 Jetty/Undertow），`main` 方法直接启动；不用再打 war 包部署到外部 Tomcat
- [ ] **良好**：及格 + 能说出启动机制——`ServletWebServerFactory`（Tomcat/Jetty/Undertow 实现）创建并启动容器、`DispatcherServlet` 自动注册、`server.port` 等配置如何生效（`ServerProperties` 绑定）
- [ ] **精通**：良好 + 能应对追问——内嵌 vs **外置部署**的取舍（内嵌：一条命令启动/版本自管；外置：运维隔离/已有 Tomcat 复用）、Tomcat 的**线程模型**（BIO/NIO，连接器线程池 + 工作线程池）与调优参数（`server.tomcat.max-threads` 等）、为什么默认端口 8080 可配置、内嵌容器如何注册 Servlet/Filter（`ServletRegistrationBean`）

**自评记录**：___（档位） 日期：___ 复答：___

---

> **通关检查**：本板块 4 题是否全部达到「精通」档？是 → 进入下一板块；否 → 标记未达标的题号：___
