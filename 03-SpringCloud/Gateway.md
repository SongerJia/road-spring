## 网关
网关是什么->为什么需要->核心概念->路由配置->过滤器->限流->和Nginx对比->面试高频题
### 网关是什么
Gateway 网关：微服务的统一入口。所有外部请求先到网关，由网关路由到具体的服务。
```
客户端 ──→ 网关（Gateway）──→ 订单服务
                  ├──→ 库存服务
                  ├──→ 用户服务
                  └──→ 支付服务
```
为什么需要网关
```
// 没有网关的问题：
// ① 客户端要知道所有服务地址（暴露内部结构）
// ② 每个服务都要做鉴权、跨域、限流（重复）
// ③ 无法统一入口控制

// 网关统一解决：
// ① 统一入口（客户端只认识网关）
// ② 统一鉴权、跨域、限流
// ③ 路由转发、负载均衡
// ④ 灰度发布、日志监控
```
### 核心概念
三大核心
```
// ① Route（路由）：一个路由 = 目标地址 + 规则
// ② Predicate（断言）：匹配条件（Path、Header 等）
// ③ Filter（过滤器）：请求/响应处理（鉴权、改写）

// 请求处理流程：
// 请求 → 匹配 Route（Predicate）→ 执行 Filter 链 → 转发到目标
```
配置示例
```
spring:
  cloud:
    gateway:
      routes:
        - id: order-service           # 路由 ID
          uri: lb://order-service     # 目标服务（lb = 负载均衡）
          predicates:                 # 断言（匹配条件）
            - Path=/api/order/**
          filters:                    # 过滤器
            - StripPrefix=2           # 去掉前 2 段路径
            - AddRequestHeader=X-Request-Id, 123  # 加请求头
```
### 核心路由配置
常用断言 Predicate
```
predicates:
  - Path=/api/order/**        # 路径匹配
  - Method=GET,POST           # 方法匹配
  - Header=X-Request-Id, \d+  # 请求头匹配（正则）
  - Query=page, \d+           # 参数匹配
  - Cookie=name, value        # Cookie 匹配
  - Host=api.example.com      # 域名匹配
  - Weight=group1, 80         # 权重（灰度）
```
常用过滤器 Filter
```
filters:
  - StripPrefix=1             # 去掉路径前缀（/api 去掉）
  - PrefixPath=/api           # 加前缀
  - AddRequestHeader=key,value    # 加请求头
  - AddResponseHeader=key,value   # 加响应头
  - SetPath=/new/path         # 重写路径
  - RewritePath=/api/(?<segment>.*), /$\{segment}  # 正则重写
  - RequestRateLimiter=...    # 限流
  - CircuitBreaker=...        # 熔断
```
完整路由示例
```
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/order/**
          filters:
            - StripPrefix=2
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=2
```
### 全局过滤器
鉴权全局过滤器
```
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    // 放行路径
    private static final Set<String> WHITELIST = Set.of(
        "/api/user/login", "/api/user/register"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // ① 获取请求路径
        String path = exchange.getRequest().getURI().getPath();

        // ② 白名单放行
        if (WHITELIST.contains(path)) {
            return chain.filter(exchange);
        }

        // ③ 校验 Token
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (token == null || !tokenService.validate(token)) {
            // 返回 401
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // ④ 传递用户信息到下游
        ServerHttpRequest request = exchange.getRequest().mutate()
            .header("X-User-Id", userId)
            .build();
        return chain.filter(exchange.mutate().request(request).build());
    }

    // 过滤器顺序（数值越小越先执行）
    @Override
    public int getOrder() {
        return -100;
    }
}
```
过滤器执行顺序
```
// ① 请求进入：GlobalFilter（全局）→ 路由 Filter → 转发
// ② 响应返回：逆序

// 执行顺序由 Ordered 决定：
// 数值小 → 先执行
// 负值 → 在所有内置过滤器前
```
### 限流
RequestRateLimiter（令牌桶）
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/order/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10    # 每秒填充 10 个令牌
                redis-rate-limiter.burstCapacity: 20    # 桶容量 20
                redis-rate-limiter.requestedTokens: 1   # 每次消耗 1 个
                key-resolver: "#{@userKeyResolver}"    # 限流维度
```
```java
// 限流维度：按用户/IP 限流
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> {
        String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
        return Mono.just(userId != null ? userId : "anonymous");
    };
}
```
### Gateway vs  Nginx vs Zuul
| 对比       | Spring Cloud Gateway | Nginx | Zuul 1.x    |
| -------- | -------------------- | ----- | ----------- |
| **类型**   | 微服务网关                | 反向代理  | 微服务网关       |
| **实现**   | WebFlux（响应式）         | C     | Servlet（阻塞） |
| **性能**   | 高                    | 最高    | 低           |
| **集成**   | 注册中心✅                | 一般    | 好           |
| **动态路由** | ✅                    | 弱     | ✅           |
| **现状**   | ✅ 主流                 | 入口层   | ⚠️ 过时       |
两者分工
```
// 生产常见架构：
// 客户端 → Nginx（入口负载均衡）→ Gateway（路由鉴权）→ 服务

// ① Nginx：最外层
//    - 域名解析、HTTPS、静态资源
//    - 入口级负载均衡

// ② Gateway：应用层
//    - 路由转发、鉴权、限流
//    - 集成注册中心、动态路由

// 面试回答：
// "Nginx 负责最外层（域名、HTTPS、静态资源），
//  Gateway 负责应用层（路由、鉴权、限流），两者配合。"
```
### 面试高频题
#### 题目1：网关的作用
```
// 统一入口、路由转发、鉴权、限流、跨域、日志
```
#### 题目2：三大核心概念
```
// Route（路由）、Predicate（断言）、Filter（过滤器）
```
#### 题目3：Gateway和Nginx的区别
```
// Nginx：入口层（域名、HTTPS、静态资源）
// Gateway：应用层（路由、鉴权、限流）
```
#### 题目4：怎么鉴权
```
// 全局过滤器（GlobalFilter）
// 校验 Token + 白名单放行 + 传递用户信息
```
#### 题目5：网关限流
```
// RequestRateLimiter（令牌桶）
// 按用户/IP 维度限流
```
#### 题目6：为什么用WebFlux不是MVC
```
// 网关是 IO 密集（转发请求）
// WebFlux 响应式：高并发、低线程占用
// 传统 Servlet 阻塞模型：每个请求占线程
```
