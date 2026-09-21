## 自动装配
自动装配是什么->原理总览->@EnableAutoConfiguration->spring.factories->条件注解->完整流程->自定义starter->面试高频题
### 自动装配是什么
自动装配：Spring Boot启动时，根据classpath中的依赖和配置文件，自动创建需要的bean，无需手动写配置。
```
// 例子：加了 spring-boot-starter-data-redis
// 自动创建：RedisConnectionFactory、RedisTemplate、StringRedisTemplate
// 不用手动 @Bean 配置

// 例子：加了 spring-boot-starter-web
// 自动创建：DispatcherServlet、内嵌 Tomcat、Jackson 转换器
```
和传统Spring相比
```
// 传统：
// <bean id="dataSource" class="..."/>
// <bean id="jdbcTemplate" class="...JdbcTemplate">
//    <property name="dataSource" ref="dataSource"/>
// </bean>

// Spring Boot：
// 加依赖 + 配置连接信息 → 自动创建
spring.datasource.url=jdbc:mysql://...
spring.datasource.username=root
```
### 原理总览
```
// 自动配置的核心：
// ① @EnableAutoConfiguration 开启
// ② spring.factories 列出所有自动配置类
// ③ 条件注解判断是否生效
// ④ 生效的配置类创建 Bean

// 流程：
@SpringBootApplication
    ↓ 包含
@EnableAutoConfiguration
    ↓ 导入
AutoConfigurationImportSelector
    ↓ 读取
META-INF/spring.factories（或 AutoConfiguration.imports）
    ↓ 逐个处理
自动配置类（DataSourceAutoConfiguration 等）
    ↓ 条件注解判断
@ConditionalOnMissingBean / @ConditionalOnClass ...
    ↓ 满足条件则
创建 Bean（注册到容器）
```
### @EnableAutoConfiguration
```
// ① 注解定义
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(AutoConfigurationImportSelector.class)  // 核心！
public @interface EnableAutoConfiguration {
    // ...
}

// ② AutoConfigurationImportSelector
// 通过 @Import 导入，负责加载所有自动配置类
public class AutoConfigurationImportSelector implements DeferredImportSelector {

    // 核心方法：返回要加载的自动配置类
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        // 读取 spring.factories 中的自动配置类
        List<String> configurations = getCandidateConfigurations(...);
        // 去重、排除（exclude）、排序
        return configurations.toArray(new String[0]);
    }

    // 读取配置：
    protected List<String> getCandidateConfigurations(...) {
        // ① Spring Boot 3.x：读取 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
        // ② Spring Boot 2.x：读取 spring.factories 中 EnableAutoConfiguration 键
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
            getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
        return configurations;
    }
}
```
为什么用DeferredImportSelector
```
// DeferredImportSelector：延迟导入
// ① 等所有普通 @Configuration 处理完后再处理自动配置
// ② 保证自动配置可以被用户配置覆盖
// ③ 支持排序（自动配置类之间也有顺序）
```
### 自动配置类
一个典型的自动配置类
```
// DataSourceAutoConfiguration 简化版
@AutoConfiguration(before = SqlInitializationAutoConfiguration.class)  // 顺序
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })   // classpath 有
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")      // 没有则配置
@EnableConfigurationProperties(DataSourceProperties.class)              // 绑定配置
public class DataSourceAutoConfiguration {

    // 条件：没有 DataSource 才创建
    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        // 根据配置创建数据源
        return properties.initializeDataSourceBuilder().build();
    }
}
```
自动配置类的构成
```
// ① @AutoConfiguration：标记是自动配置类（3.0+）
// ② @ConditionalOnXxx：生效条件
// ③ @EnableConfigurationProperties：绑定 application.yml 配置
// ④ @Bean：创建 Bean
```
### 条件注解
|注解|条件|例子|
|---|---|---|
|**@ConditionalOnClass**|classpath 有某类|有 Redis 类才配置|
|**@ConditionalOnMissingBean**|容器没有某 Bean|用户自定义了就不配置|
|**@ConditionalOnBean**|容器有某 Bean|有 DataSource 才配置 JdbcTemplate|
|**@ConditionalOnProperty**|配置项存在/匹配|开关配置|
|**@ConditionalOnExpression**|SpEL 表达式|复杂条件|
|**@ConditionalOnWebApplication**|是 Web 应用|MVC 相关配置|
|**@ConditionalOnMissingClass**|classpath 没有某类|兜底|
为什么条件注解重要
```
// ① 按需生效：没有依赖就不创建（省内存）
// ② 可覆盖：用户自定义的 Bean 优先（@ConditionalOnMissingBean）
// ③ 可配置：@ConditionalOnProperty 开关

// 例子：Redis 自动配置
@ConditionalOnClass(RedisOperations.class)   // 有 Redis 依赖才生效
@ConditionalOnMissingBean(RedisTemplate.class)  // 用户没自定义才创建
@ConditionalOnProperty(prefix = "spring.data.redis", name = "enabled", matchIfMissing = true)  // 开关
public class RedisAutoConfiguration { ... }
```
条件注解的解析时机
```
// 条件在 ConfigurationClassPostProcessor 处理时判断
// （refresh 的 invokeBeanFactoryPostProcessors 阶段）
// 不满足条件的配置类被跳过（不解析 @Bean）
```
### 配置绑定（@ConfigurationProperties）
把yml绑定到对象
```
// application.yml：
// spring:
//   datasource:
//     url: jdbc:mysql://localhost:3306/db
//     username: root
//     password: 123456

// 绑定：
@Data
@Component
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties {
    private String url;
    private String username;
    private String password;
}

// 用法一：@EnableConfigurationProperties(DataSourceProperties.class)
// 用法二：@Component + @ConfigurationProperties
// 用法三：@Bean + @ConfigurationProperties

// 松散绑定：
// url → url（严格）
// user-name → userName（kebab-case → camelCase）✅
// USER_NAME → userName（大写）✅
```
### 完整流程串联
```
启动（@SpringBootApplication）
    ↓
@EnableAutoConfiguration
    ↓
AutoConfigurationImportSelector.selectImports
    ↓
读取 spring.factories / AutoConfiguration.imports
    ↓
得到所有自动配置类（如 RedisAutoConfiguration）
    ↓
DeferredImportSelector 延迟处理（等用户配置先处理）
    ↓
条件注解判断（@ConditionalOnClass 等）
    ↓
满足条件的自动配置类生效
    ↓
@EnableConfigurationProperties 绑定配置
    ↓
@Bean 创建 Bean → 注册到容器
    ↓
用户可覆盖（@ConditionalOnMissingBean）
```
### 自定义starter
```
// 需求：写一个自己的 starter（如：短信发送自动配置）

// ① 定义自动配置类
@AutoConfiguration
@ConditionalOnClass(SmsSender.class)
@EnableConfigurationProperties(SmsProperties.class)
public class SmsAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public SmsSender smsSender(SmsProperties properties) {
        return new SmsSender(properties.getAccessKey(), properties.getSecretKey());
    }
}

// ② 配置属性
@ConfigurationProperties(prefix = "sms")
public class SmsProperties {
    private String accessKey;
    private String secretKey;
}

// ③ 注册自动配置类（META-INF/spring/...AutoConfiguration.imports）：
// com.example.SmsAutoConfiguration

// ④ 用户使用：
// application.yml：
// sms:
//   access-key: xxx
//   secret-key: xxx

// 依赖中加入 starter → 自动装配 SmsSender ✅
```
### 面试高频题
### 题目1：自动装配的原理
```
// @EnableAutoConfiguration → AutoConfigurationImportSelector
// 读取 spring.factories/AutoConfiguration.imports
// 条件注解判断 → 生效的创建 Bean
```
#### 题目2：自动装配类在哪
```
// Spring Boot 2.x：META-INF/spring.factories
// Spring Boot 3.x：META-INF/spring/...AutoConfiguration.imports
```
#### 题目3：怎么覆盖自动配置
```
// ① 自己 @Bean 同名/同类型（@ConditionalOnMissingBean）
// ② exclude 排除自动配置类
// ③ @ConditionalOnProperty 关掉
```
#### 题目4：条件注解有哪些
```
// @ConditionalOnClass/MissingBean/Bean/Property/Expression...
```
#### 题目5：@SpringBootApplication的自动装配入口
```
// @EnableAutoConfiguration
```
#### 题目6：自动配置为什么不生效
```
// ① 依赖没加（@ConditionalOnClass 不满足）
// ② 被 exclude 排除
// ③ 用户已自定义 Bean（MissingBean 不满足）
// ④ 配置开关关闭（OnProperty）
```
