## 配置中心
为什么需要配置中心->配置中心是什么->Nacos config使用->动态刷新->配置优先级->面试高频题
### 为什么需要配置中心
传统配置文件的问题
```
// ① 修改配置要重新发布（重启服务）
// ② 每个服务一份配置（改了要挨个改）
// ③ 多环境切换麻烦（dev/prod 手动改）
// ④ 敏感信息（密码）散落在代码/文件里
// ⑤ 无法回滚配置
```
场景
```
// 线上开关：临时关闭某个功能
// 改配置 → 重启 → 影响用户！
// 配置中心：改配置 → 自动刷新 → 无重启
```
### 配置中心是什么
配置中心：集中管理所有微服务的配置，支持动态修改、实时刷新和版本管理。
```
配置中心（Nacos Config）
    │
    ├── 订单服务（启动拉取 + 订阅）
    ├── 库存服务（启动拉取 + 订阅）
    └── 用户服务（启动拉取 + 订阅）

配置修改 → 通知各服务 → 动态刷新（无需重启）
```

|配置中心|特点|现状|
|---|---|---|
|**Nacos Config**|注册+配置一体|✅ 国内主流|
|**Apollo**|携程开源，功能强|可用|
|**Spring Cloud Config**|老牌，Git 存储|一般|
推荐Nacos原因
```
// ① 注册中心 + 配置中心二合一（一套基础设施）
// ② 控制台友好（可视化编辑）
// ③ 支持动态刷新、灰度、命名空间隔离
// ④ 国内生态完善
```
### Nacos config使用
依赖与配置
```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```
```yaml
# bootstrap.yml（引导配置，先加载）
spring:
  application:
    name: order-service
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yml          # 配置格式
        group: DEFAULT_GROUP
        namespace: dev               # 命名空间（环境隔离）
```
配置存放
```
Nacos 控制台 → 配置管理 → 新建配置：
Data ID: order-service.yml      ← 服务名 + 扩展名
Group: DEFAULT_GROUP
格式: YAML

内容：
server:
  port: 8080
order:
  timeout: 3000
```
代码使用
```
// 和 application.yml 一样注入
@Value("${order.timeout}")
private int timeout;

// 或批量绑定
@Data
@Component
@ConfigurationProperties(prefix = "order")
public class OrderProperties {
    private int timeout;
}
```
### 动态刷新
@RefreshScope
```
// 配置变更后自动刷新
@RefreshScope  // 关键注解！
@Component
@ConfigurationProperties(prefix = "order")
public class OrderProperties {
    private int timeout;
}

// 原理：
// @RefreshScope 的 Bean 是"动态代理"
// 配置变更 → 刷新事件 → 重新创建 Bean（重新绑定配置）
```
刷新流程
```
Nacos 控制台修改配置
    ↓
Nacos 服务端保存新配置
    ↓
推送变更通知（长轮询/UDP）
    ↓
客户端感知变更
    ↓
发布 RefreshEvent
    ↓
@RefreshScope 的 Bean 重新创建
    ↓
新配置生效（无需重启）
```
不生效的场景
```
// ① 没加 @RefreshScope → 不刷新
// ② @Value 注入的字段 → 加 @RefreshScope 的类才能刷
// ③ 静态字段 → 不刷新
// ④ 常量（final）→ 不刷新
```
### 配置优先级
Nacos与本地配置的优先级
```
// 优先级（高 → 低）：
// ① 命令行参数
// ② Nacos 远程配置
// ③ 本地 application.yml
// ④ 默认值

// 结论：Nacos 配置 > 本地配置
// （远程配置可以覆盖本地）
```
多个 Data ID的加载
```
// ① order-service.yml（主配置）
// ② order-service-dev.yml（环境配置）
// ③ extension-configs（扩展配置）
// ④ shared-configs（共享配置）

// 优先级：扩展 > 主配置 > 共享
```
### 命名空间与分组
环境隔离
```
# 命名空间（namespace）：隔离环境
# dev / test / prod 用不同命名空间
spring:
  cloud:
    nacos:
      config:
        namespace: dev  # 命名空间 ID
```
配置隔离
```
// ① 命名空间：环境隔离（dev/prod）
// ② 分组（group）：同一环境下按业务分
// ③ Data ID：具体配置

// 场景：
// dev 环境的配置改坏 → 不影响 prod ✅
```
### 面试高频题
#### 题目1：为什么需要配置中心
```
// 改配置不重启、统一管理、多环境、版本回滚
```
#### 题目2：Nacos config怎么动态刷新
```
// 修改 → 通知 → @RefreshScope 重新创建 Bean
```
#### 题目3：配置中心的优先级
```
// 命令行 > Nacos > 本地 yml
```
#### 题目4：动态刷新不生效怎么办
```
// ① 加 @RefreshScope
// ② 检查 namespace/group
// ③ 检查文件格式
```
#### 题目5：敏感配置怎么处理
```
// ① Nacos 支持加密（Jasypt）
// ② 密钥放环境变量
// ③ 权限控制
```
#### 题目6：配置中心挂了怎么办
```
// ① 本地缓存（客户端保留最后配置）
// ② 启动时拉不到 → 用本地配置兜底
// ③ 配置中心集群高可用
```


