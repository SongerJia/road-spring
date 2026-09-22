## 启动流程
SpringBoot是什么->启动入口->run()->启动流程详解->和Spring启动区别->面试高频题
### Spring Boot是什么
Spring Boot：简化Spring开发的框架，核心是约定大于配置，自动配置+starter。
解决什么问题
```
// 传统 Spring 的痛点：
// ① 大量 XML 配置（数据源、事务、MVC...）
// ② 依赖版本冲突（jar 兼容）
// ③ 部署复杂（要装 Tomcat）

// Spring Boot 解决：
// ① 自动配置（不用写配置）
// ② starter 统一依赖管理
// ③ 内嵌容器（java -jar 直接跑）
```
### 启动入口
```
@SpringBootApplication  // 组合注解！
public class JudgeApplication {
    public static void main(String[] args) {
        // 启动入口
        SpringApplication.run(JudgeApplication.class, args);
    }
}
```
@SpringBootApplication是什么
```
// @SpringBootApplication = 三个注解的组合：
// ① @SpringBootConfiguration   —— 配置类（内含 @Configuration）
// ② @EnableAutoConfiguration    —— 开启自动配置（核心！）
// ③ @ComponentScan              —— 包扫描（扫描本类所在包及子包）

// 注意：
// 主类要放在包的根目录（保证扫描范围正确）
```
SpringApplicaiton.run()做了什么
```
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    return new SpringApplication(primarySource).run(args);
}

两步
// ① new SpringApplication()：初始化
// ② run()：启动流程
```
### 启动流程详解
run()的完整流程
```
public ConfigurableApplicationContext run(String... args) {

    // ① 启动计时器（记录启动耗时）
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();

    // ② 引导上下文（创建 BootstrapContext）
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();

    // ③ 设置应用上下文类型（Servlet/Reactive）
    // 根据 classpath 判断：Web 应用 / WebFlux / 普通

    // ④ 加载 ApplicationContextInitializer（初始化器）
    // 从 spring.factories 加载

    // ⑤ 加载 ApplicationListener（应用监听器）
    // 从 spring.factories 加载

    // ⑥ 找出主类（main 方法所在类）
    // 从栈信息推断

    // ⑦ 启动环境准备
    ConfigurableEnvironment environment = prepareEnvironment(...);
    // 加载 application.yml、系统属性、环境变量
    // 启动配置日志（版本 banner 等）

    // ⑧ 打印 Banner（启动图案）

    // ⑨ 创建 ApplicationContext（上下文）
    context = createApplicationContext();
    // AnnotationConfigServletWebServerApplicationContext（Web）
    // 或 AnnotationConfigApplicationContext（普通）

    // ⑩ 准备上下文
    prepareContext(bootstrapContext, context, environment, ...);
    // ① 注册 BeanNameGenerator
    // ② 应用初始化器（ApplicationContextInitializer）
    // ③ 注册主类（作为 Bean）
    // ④ 加载资源（主类所在包的 BeanDefinition）
    // ⑤ 发布 ApplicationPreparedEvent

    // ⑪ 刷新上下文 ★ 核心！
    refreshContext(context);
    // 调用 Spring 的 refresh() 方法（前面挖过 12 步）
    // 实例化所有单例 Bean

    // ⑫ 刷新后处理
    afterRefresh(context, applicationArguments);
    // 调用 CommandLineRunner / ApplicationRunner

    // ⑬ 停止计时，打印启动耗时
    stopWatch.stop();

    // ⑭ 发布 ApplicationReadyEvent（启动完成事件）
    // 此时应用可以接收请求了

    return context;
}

① 准备环境（加载配置）
② 创建上下文（ApplicationContext）
③ 准备上下文（初始化器、主类）
④ 刷新上下文（= Spring refresh 12 步）★
⑤ 执行 Runner（CommandLineRunner）
⑥ 发布就绪事件（启动完成）
```
### refreshContext之后
刷新上下文
```
// refreshContext 调用的是 Spring 的 refresh()（12 步）
// 但 Spring Boot 通过子类增强了它：
// ① 自动配置在 invokeBeanFactoryPostProcessors 阶段加载
// ② 内嵌 Web 容器在 onRefresh() 阶段启动（Tomcat）
// ③ WebServerStartStopLifecycle 管理容器生命周期
```
Runner（启动后执行）
```
// 应用启动完成后要执行的逻辑：

// 方式一：CommandLineRunner（原始参数）
@Component
public class StartupRunner implements CommandLineRunner {
    @Override
    public void run(String... args) {
        // 缓存预热、初始化数据
        System.out.println("启动完成，预热缓存...");
    }
}

// 方式二：ApplicationRunner（封装参数）
@Component
public class StartupRunner2 implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        // args.getOptionValues("key") 方便取参数
    }
}

// 多个 Runner 排序：
@Component
@Order(1)
public class Runner1 implements CommandLineRunner { ... }
```
就绪事件
```
// ApplicationReadyEvent：应用完全就绪
// 此时所有 Bean 创建完成，可以接收请求

// 监听（初始化完成后的逻辑）：
@Component
public class ReadyListener {
    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        // 定时任务启动、预热
    }
}
```
### 启动流程 vs Spring
| 步骤     | 传统 Spring | Spring Boot     |
| ------ | --------- | --------------- |
| 配置     | XML 手动    | 自动配置            |
| 容器     | 手动创建      | run() 自动        |
| Web 容器 | 外部 Tomcat | 内嵌              |
| 配置文件   | 多个 XML    | application.yml |
| 启动     | 复杂        | 一行 main         |
```
面试官："Spring Boot 启动流程？"

"核心是 SpringApplication.run()：
① 准备环境（加载 application.yml、系统属性）
② 创建 ApplicationContext
③ 准备上下文（加载初始化器、注册主类）
④ 刷新上下文：执行 Spring 的 refresh()（自动配置在这里生效，
   内嵌 Tomcat 在 onRefresh 启动）
⑤ 执行 CommandLineRunner/ApplicationRunner
⑥ 发布 ApplicationReadyEvent，应用就绪

其中最关键的是刷新上下文那步，
自动配置和 Bean 创建都在里面。"
```
### 面试高频题
#### 题目1：@SpringBootApplication是什么
```
// 组合注解：@SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan
```
#### 题目2：run()的核心流程是什么
```
// 环境 → 上下文 → 刷新 → Runner → 就绪
```
#### 题目3：启动完之后怎么执行逻辑
```
// CommandLineRunner / ApplicationRunner / @EventListener(ApplicationReadyEvent)
```
#### 题目4：自动配置在哪一步生效
```
// refresh() 的 invokeBeanFactoryPostProcessors 阶段
// ConfigurationClassPostProcessor 处理 @EnableAutoConfiguration
```
#### 题目5：内嵌tomcat什么时候启动
```
// refresh() 的 onRefresh() 阶段
// WebServerFactory 创建 Tomcat 并启动
```
#### 题目6：怎么拿到启动参数
```
// ApplicationRunner 的 ApplicationArguments
// 或 @Value 注入
```
