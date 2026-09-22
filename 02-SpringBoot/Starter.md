## Starter机制
Starter是什么->常用starter->starter原理->自定义starter->面试高频题
### Starter是什么
Starter：把某个功能的依赖和自动配置打包成一个开箱即用的模块，加一个依赖，功能就自动可用。
```
<!-- 加 Web starter → MVC + Tomcat + Jackson 自动配置好 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- 加 Redis starter → RedisTemplate 自动创建 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
starter解决了什么
```
// ① 依赖版本管理：不用自己协调版本（父 POM 统一）
// ② 自动配置：不用写配置类
// ③ 开箱即用：加依赖就可用
// ④ 减少配置错误
```
### 常用的starter
```
//web相关
spring-boot-starter-web          <!-- MVC + Tomcat -->
spring-boot-starter-webflux      <!-- 响应式 -->
spring-boot-starter-websocket    <!-- WebSocket -->

//数据相关
spring-boot-starter-data-redis   <!-- Redis -->
spring-boot-starter-data-jpa     <!-- JPA -->
spring-boot-starter-jdbc         <!-- JDBC -->
spring-boot-starter-validation   <!-- 参数校验 -->

//其他
spring-boot-starter-security     <!-- 安全 -->
spring-boot-starter-actuator     <!-- 监控 -->
spring-boot-starter-test         <!-- 测试 -->
spring-boot-starter-logging      <!-- 日志 -->
spring-boot-starter-aop          <!-- AOP -->
spring-boot-starter-mail         <!-- 邮件 -->
```
starter命名规则
```
// Spring 官方：spring-boot-starter-xxx
// 第三方：xxx-spring-boot-starter（如 mybatis-spring-boot-starter）
```
### starter原理
一个starter结构
```
// spring-boot-starter-data-redis 包含：
// ① 依赖：redis 客户端（Lettuce）
// ② 自动配置：RedisAutoConfiguration
// ③ 配置属性：RedisProperties
// ④ 注册文件：AutoConfiguration.imports（列出自动配置类）
```
核心流程
```
加依赖（starter）
    ↓
classpath 有了 Redis 相关类
    ↓
@EnableAutoConfiguration 读取 AutoConfiguration.imports
    ↓
找到 RedisAutoConfiguration
    ↓
@ConditionalOnClass(RedisOperations) 满足（有类）
    ↓
创建 RedisTemplate 等 Bean
    ↓
自动配置完成 ✅
```
starter=依赖+自动配置类+注册文件
```
starter 包结构：
src/main/resources/META-INF/spring/
    └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
        （列出所有自动配置类，一行一个）
```
### 自定义starter
需求：做一个 短信发送 Starter
项目结构
```
sms-spring-boot-starter/
├── pom.xml
└── src/main/java/com/example/sms/
    ├── SmsSender.java              （核心类）
    ├── SmsProperties.java          （配置属性）
    ├── SmsAutoConfiguration.java   （自动配置类）
    └── resources/META-INF/spring/
        └── AutoConfiguration.imports
```
核心类
```
// SmsSender：核心功能
public class SmsSender {
    private final String accessKey;
    private final String secretKey;

    public SmsSender(String accessKey, String secretKey) {
        this.accessKey = accessKey;
        this.secretKey = secretKey;
    }

    public void send(String phone, String content) {
        System.out.println("发送短信到 " + phone + ": " + content);
    }
}

// SmsProperties：配置属性
@ConfigurationProperties(prefix = "sms")
public class SmsProperties {
    private String accessKey;
    private String secretKey;
    // getter/setter
}

// SmsAutoConfiguration：自动配置类
@AutoConfiguration
@ConditionalOnClass(SmsSender.class)           // 有类才生效
@EnableConfigurationProperties(SmsProperties.class)  // 绑定配置
public class SmsAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                    // 用户没自定义才创建
    @ConditionalOnProperty(prefix = "sms", name = "enabled", havingValue = "true", matchIfMissing = true)
    public SmsSender smsSender(SmsProperties properties) {
        return new SmsSender(properties.getAccessKey(), properties.getSecretKey());
    }
}
```
注册自动配置类
```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.sms.SmsAutoConfiguration
```
用户使用
```
<dependency>
    <groupId>com.example</groupId>
    <artifactId>sms-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>

sms:
  access-key: xxx
  secret-key: xxx

@Autowired
private SmsSender smsSender;  // 自动注入 ✅

smsSender.send("13800000000", "验证码 1234");
```
### Starter细节
条件注解的完整组合
```
// 一个好的自动配置类应该：
// ① @ConditionalOnClass：依赖存在才生效
// ② @ConditionalOnMissingBean：用户可覆盖
// ③ @ConditionalOnProperty：可开关
// ④ @AutoConfiguration(before/after)：控制顺序
```
配置提示
```
<!-- 可选：生成配置元数据（IDE 写 yml 有提示） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```
常见问题
```
// ① 自动配置不生效：imports 文件路径/名字写错
// ② 条件不满足：依赖没加
// ③ 和用户 Bean 冲突：@ConditionalOnMissingBean
// ④ 配置不提示：加 configuration-processor
```
### 面试高频题
#### 题目1：starter是什么？原理？
```
// 依赖 + 自动配置打包
// 加依赖 → 自动配置类生效 → 创建 Bean
```
#### 题目2：怎么自定义starter
```
// ① 核心类 + Properties
// ② 自动配置类（条件注解）
// ③ imports 文件注册
// ④ 用户加依赖 + 配置即可
```
#### 题目3：starter和自动配置关系
```
// starter 提供依赖和自动配置
// 自动配置是 starter 的核心
// 没有 starter 也能手动加依赖 + 自动配置
```
#### 题目4：为什么加依赖就生效
```
// @EnableAutoConfiguration 扫描 imports 文件
// 条件注解判断（有依赖的类 → 生效）
```
#### 题目5：自动配置怎么被用户覆盖
```
// @ConditionalOnMissingBean：用户自定义了就不创建
// 这是"约定大于配置 + 可覆盖"的关键
```

