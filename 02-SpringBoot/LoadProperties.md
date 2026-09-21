## 配置加载
配置文件->优先级顺序->多环境配置->配置绑定->动态配置->面试高频题
### 配置文件
SpringBoot支持两种格式
```yml
# application.yml（推荐）
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/db
```
```properties
# application.properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/db
```
两种格式优先级
```
// 如果同时存在：
// application.properties > application.yml
// properties 优先级更高（先加载）

// 但实际项目：用 yml（可读性更好），不要混用
```
### 配置文件优先级
配置文件位置
```
// Spring Boot 按优先级加载配置文件： 优先级从高到低
// ① file:./config/          —— 项目根目录的 config 目录
// ② file:./                 —— 项目根目录
// ③ classpath:/config/      —— classpath 的 config 目录
// ④ classpath:/             —— classpath 根目录

// 高优先级覆盖低优先级（同名配置）

// 另外：命令行参数 > 系统属性 > 环境变量 > 配置文件
```
完整优先级链
```
// 配置来源优先级（高 → 低）：
// ① 命令行参数：--server.port=9090
// ② Java 系统属性：-Dserver.port=9090
// ③ 操作系统环境变量：SERVER_PORT=9090
// ④ application-{profile}.yml（特定环境）
// ⑤ application.yml（主配置）
// ⑥ 默认值

// 命令行最优先：
java -jar app.jar --server.port=9090 --spring.profiles.active=prod
```
### 多环境配置
按环境分离
```
# application.yml（公共配置）
server:
  port: 8080

# application-dev.yml（开发）
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/dev

# application-prod.yml（生产）
spring:
  datasource:
    url: jdbc:mysql://prod-db:3306/prod
```
激活环境
```
# ① 配置文件指定
# application.yml：
# spring:
#   profiles:
#     active: dev

# ② 命令行
java -jar app.jar --spring.profiles.active=prod

# ③ 环境变量
export SPRING_PROFILES_ACTIVE=prod
```
profile的优先级
```
// application-{profile}.yml 优先于 application.yml
// 特定环境的配置会覆盖公共配置
```
### 配置绑定
@Value简单取值
```
@Component
public class ConfigReader {
    @Value("${server.port}")
    private int port;

    @Value("${app.name:默认名}")  // 带默认值
    private String appName;

    @Value("${spring.datasource.url}")
    private String dbUrl;
}
```
@ConfigurationProperties
```
@Data
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private int maxThreads;
    private List<String> domains = new ArrayList<>();
    private Map<String, String> headers = new HashMap<>();
    private Nested nested = new Nested();  // 嵌套对象

    @Data
    public static class Nested {
        private String key;
        private String secret;
    }
}

// application.yml：
// app:
//   name: judge-platform
//   max-threads: 100
//   domains:
//     - a.com
//     - b.com
//   nested:
//     key: xxx
//     secret: xxx

// 好处：类型安全、支持复杂结构、自动校验
```

|对比|@Value|@ConfigurationProperties|
|---|---|---|
|**绑定**|单个字段|批量对象|
|**类型安全**|弱|✅ 强|
|**复杂结构**|❌ 难|✅ 支持|
|**校验**|❌|✅ @Validated|
|**松绑定**|❌ 严格|✅ kebab-case 支持|
|**推荐**|简单取值|✅ 配置对象|
### 配置校验与刷新
配置校验
```
@Data
@Component
@ConfigurationProperties(prefix = "app")
@Validated  // 开启校验
public class AppProperties {
    @NotBlank
    private String name;

    @Min(1)
    @Max(1000)
    private int maxThreads;

    @Pattern(regexp = "^[a-zA-Z0-9]+$")
    private String token;
}
```
动态刷新
```
// Spring Cloud Config / Nacos 支持配置热更新

// 方式一：@RefreshScope（重新创建 Bean）
@RefreshScope
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties { ... }

// 配置变更 → 通知 → Bean 重新绑定

// 方式二：@ConfigurationProperties + @ConfigurationPropertiesScan
// Nacos 的 dataId 变更自动刷新
```
### 面试高频题
#### 题目1：配置文件的优先级
```
// 命令行 > 系统属性 > 环境变量 > profile > 主配置
// 位置：config/ > 根目录 > classpath/config > classpath
```
#### 题目2：多环境怎么配置
```
// application-{profile}.yml + spring.profiles.active
```
#### 题目3：@Vaule和@ConfigurationProperties
```
// @Value：单个值
// @ConfigurationProperties：批量对象（推荐）
```
#### 题目4：生产环境配置怎么管理
```
// ① 敏感信息用环境变量/密钥管理
// ② 配置中心（Nacos）统一管理 + 热更新
// ③ 加密敏感配置
```
#### 题目5：松绑定是什么
```
// app.max-threads → maxThreads（kebab-case → camelCase）
// 配置文件用 kebab-case，代码用 camelCase
```
#### 题目6：配置不生效排查
```
// ① 前缀写错
// ② 没加 @ConfigurationProperties 扫描
// ③ 多环境覆盖
// ④ 缓存（配置中心需刷新）
```
